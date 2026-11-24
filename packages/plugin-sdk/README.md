# @sadshachatdesk/plugin-sdk

SDK cho plugin SadSha ChatDesk.

- **UI plugin (iframe):** không cần cài gì. Thêm `<script src="/__sadshachatdesk/sdk.js"></script>` vào trang của plugin; SadSha ChatDesk phục vụ đúng phiên bản khớp với app. File `web.js` ở đây là bản sao cho bundler hoặc UI chạy trên server ngoài (plugin remote).
- **Giao diện:** tự do — HTML/CSS/JS thuần, Tailwind, React, Vue, Svelte… (build ra file tĩnh trong thư mục UI của plugin), CSS hoàn toàn của bạn. Muốn hợp theme sáng/tối của SadSha ChatDesk thì đọc `sadshachatdesk.ui.theme()` / `onTheme()` hoặc dùng các biến CSS `--dp-*` mà SDK gắn sẵn lên trang (tuỳ chọn).
- **Backend (runtime `local`):** nhận `ctx.sadshachatdesk` trong `activate`, `nodes`, `routes`, `handlers`, `commands`. Kiểu xem `index.d.ts` (`PluginBackend`, `BackendContext`).

```js
/** @type {import('@sadshachatdesk/plugin-sdk').PluginBackend} */
module.exports = {
  async activate({ sadshachatdesk, settings }) {
    sadshachatdesk.events.on('message.received', (ev) => console.log(ev.data.message.text));
  },
  nodes: {
    'shop.hello': async (cfg, run, { sadshachatdesk }) => {
      await sadshachatdesk.chat.sendMessage({ accountId: run.pageId, threadId: cfg.threadId, text: 'Xin chào!' });
      return { ok: true };
    },
  },
};
```

## Quy tắc

- Tên hàm API = `operationId` trong `/api/v1/openapi.json` (xem `/docs`). Nhóm (`chat`, `crm`…) chỉ để dễ đọc.
- Mọi lời gọi đi qua API v1 với principal `plugin:<id>`: chỉ làm được trong phạm vi quyền đã duyệt khi cài, chỉ trên tài khoản được cấp.
- Hàm riêng của plugin: `storage`, `settings`, `log`, `events`, `workflow.trigger`, `media.upload`, `system.info`, `backend.call` (UI), `http.fetch` (backend, cần quyền `network`; trả `text` hoặc `base64`, ép bằng `init.responseType`), `cloudflare` (backend, cần quyền `cloudflare:workers`).
- Hàm API trả file (`downloadMessageMedia`, `downloadLibraryItem`, `downloadLibraryThumb`) trả `{ name, mime, size, base64 }` (tối đa 25 MB, cần quyền `media:read`). Ghi âm Zalo chưa có trên máy thì SadSha ChatDesk tự tải bằng phiên của tài khoản.

## Triển khai lên Cloudflare (`sadshachatdesk.cloudflare`)

Ứng dụng có quyền `cloudflare:workers` (runtime `local`) tạo được Worker và cơ sở dữ liệu D1 trên tài khoản Cloudflare mà người dùng đã kết nối trong SadSha ChatDesk (Cài đặt → Internet & Cloud). Token Cloudflare không bao giờ đưa cho ứng dụng: SadSha ChatDesk kiểm quyền và gọi Cloudflare thay.

- Tên Worker / D1 phải **đúng bằng tiền tố** của ứng dụng (phần cuối id: `io.github.acme.fintrack` → `fintrack`) hoặc **bắt đầu bằng `<tiền tố>-`** (`fintrack-dev`); tên khác → lỗi `CF_NAME_FORBIDDEN`.
- `bundleDir` / thư mục migration là đường dẫn tương đối trong thư mục ứng dụng đã cài; gói theo dạng `bundle.json` + `worker/` + `assets/` (tuỳ chọn). Tổng ≤ 50 MB, ≤ 5000 file asset, mỗi file ≤ 25 MiB.
- Token thiếu quyền → `CF_PERMISSION` (message nêu quyền cần thêm: Workers Scripts · Edit, D1 · Edit); chưa kết nối → `CF_NOT_CONNECTED`; Cloudflare báo lỗi → `CF_API`.
- Mỗi lời gọi tốn request trong ngân sách 100 request / 5 phút dùng chung của SadSha ChatDesk. Triển khai có thể mất vài phút: chạy ở nền và ghi tiến độ bằng `storage`, đừng để UI chờ `backend.call`.
- Tên miền riêng cho Worker: `sadshachatdesk.cloudflare.zones()` liệt kê tên miền của tài khoản; `workers.domains.attach(name, 'app.tenmien.vn')` gắn tên con (Cloudflare tự tạo bản ghi DNS + chứng chỉ, vài phút sau tên mới trả lời được qua HTTPS); `workers.domains.list(name)`, `workers.domains.detach(name, hostname)`. Chỉ gắn tên con mới: không gắn tên miền gốc, không đè bản ghi DNS đang có hay tên đang gắn cho Worker khác (`CONFLICT`). Trước khi ghi, SadSha ChatDesk hỏi người dùng bằng hộp thoại của SadSha ChatDesk; bấm "Không" → `CF_DECLINED`.

```js
const crypto = require('crypto');

async function getOrCreateSecret(sadshachatdesk) {        // cần thêm quyền storage
  let secret = await sadshachatdesk.storage.get('integrationSecret');
  if (!secret) {
    secret = crypto.randomBytes(32).toString('base64url');
    await sadshachatdesk.storage.set('integrationSecret', secret);
  }
  return secret;
}

/** @type {import('@sadshachatdesk/plugin-sdk').PluginBackend} */
module.exports = {
  commands: {
    async deploy(_params, { sadshachatdesk }) {
      const st = await sadshachatdesk.cloudflare.status();
      if (!st.connected) return { error: 'Hãy kết nối Cloudflare trong SadSha ChatDesk trước' };
      if (st.missing.length) return { error: `Token thiếu quyền: ${st.missing.join(', ')}` };

      await sadshachatdesk.cloudflare.d1.ensure(st.prefix);                                   // vd "fintrack"
      await sadshachatdesk.cloudflare.d1.migrate(st.prefix, 'vendor/fintrack/migrations');
      const res = await sadshachatdesk.cloudflare.workers.deployBundle({
        name: st.prefix,
        bundleDir: 'vendor/fintrack',
        bindings: { d1: { DB: st.prefix }, ai: 'AI', vars: { APP_ENV: 'production' } },
        secrets: { SADSHA_CHATDESK_INTEGRATION_SECRET: await getOrCreateSecret(sadshachatdesk) },
      });
      return { url: res.url };
    },
  },
};
```

Đồng bộ `web.js` sau khi sửa SDK trong app: `node scripts/sync-plugin-sdk.js`.
