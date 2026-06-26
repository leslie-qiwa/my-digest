# Captcha proves you're human. HATCHA proves you're not

CAPTCHA 证明你是人类。HATCHA 证明你不是。

HATCHA(超高速智能体计算启发式评估测试,Hyperfast Agent Test for Computational Heuristic Assessment)是一种反向验证码,它通过对 AI 智能体来说轻而易举、但对人类来说很痛苦的挑战来限制访问——大数乘法、字符串反转、二进制解码等。

- 服务端验证——答案绝不会到达客户端。HMAC 签名令牌,无状态,无需数据库。
- 5 种内置挑战类型——数学、字符串反转、字符计数、排序、二进制解码。
- 可扩展——在运行时注册自定义挑战生成器。
- 可主题化——通过 CSS 自定义属性实现深色、浅色或自动模式。
- 框架适配器——开箱即用地支持 Next.js App Router 和 Express 中间件。

npm install @mondaycom/hatcha-react @mondaycom/hatcha-server

```ts
// app/api/hatcha/[...hatcha]/route.ts
import { createHatchaHandler } from "@mondaycom/hatcha-server/nextjs";
const handler = createHatchaHandler({
secret: process.env.HATCHA_SECRET!,
});
export const GET = handler;
export const POST = handler;
```

```tsx
// app/layout.tsx
import { HatchaProvider } from "@mondaycom/hatcha-react";
import "@mondaycom/hatcha-react/styles.css";
export default function RootLayout({ children }) {
return (
<html lang="en">
<body>
<HatchaProvider>{children}</HatchaProvider>
</body>
</html>
);
}
```

```tsx
"use client";
import { useHatcha } from "@mondaycom/hatcha-react";
function AgentModeButton() {
const { requestVerification } = useHatcha();
return (
<button
onClick={() =>
requestVerification((token) => {
console.log("Agent verified!", token);
})
}
>
Enter Agent Mode
</button>
);
}
```

```
# .env.local
HATCHA_SECRET=your-random-secret-here
```

```
客户端                                服务端
│ │
│ GET /api/hatcha/challenge │
│────────────────────────────────►│
│ │ 生成挑战
│ │ 哈希处理答案
│ │ HMAC 签名 { hash, expiry }
│ { challenge(不含答案), token } │
│◄────────────────────────────────│
│ │
│ 智能体求解该挑战 │
│ │
│ POST /api/hatcha/verify │
│ { answer, token } │
│────────────────────────────────►│
│ │ 验证 HMAC 签名
│ │ 检查过期时间
│ │ 比对答案哈希
│ { success, verificationToken } │
│◄────────────────────────────────│
```

答案绝不会到达客户端。已签名的令牌是不透明的,仅包含一个经哈希处理的答案 + 过期时间。验证是无状态的——无需数据库。

| 类型 | 图标 | 功能 | 时间限制 |
|---|---|---|---|
| `math` | × | 5 位数 × 5 位数乘法 | 30 秒 |
| `string` | ↔ | 反转一个 60–80 个字符的随机字符串 | 30 秒 |
| `count` | # | 在约 250 个字符中统计某个特定字符 | 30 秒 |
| `sort` | ⇅ | 对 15 个数字排序,返回第 k 小的值 | 30 秒 |
| `binary` | 01 | 将二进制字节解码为 ASCII | 30 秒 |

```ts
import { registerChallenge } from "@mondaycom/hatcha-server";
registerChallenge({
type: "hex",
generate() {
const n = Math.floor(Math.random() * 0xffffff);
return {
display: {
type: "hex",
icon: "0x",
title: "Hex Decode",
description: "Convert this hex number to decimal.",
prompt: `0x${n.toString(16).toUpperCase()}`,
timeLimit: 30,
answer: String(n),
},
answer: String(n),
};
},
});
```

HATCHA 使用作用域限定在 `--hatcha-*` 下的 CSS 自定义属性。可在任意父元素上覆盖它们:

```css
[data-hatcha-theme] {
--hatcha-accent: #3b82f6;
--hatcha-accent-light: #60a5fa;
--hatcha-bg: #060b18;
--hatcha-fg: #e4eaf6;
--hatcha-success: #22c55e;
--hatcha-danger: #ef4444;
}
```

向 `<HatchaProvider>` 或 `<Hatcha>` 传入 `theme="dark"`、`theme="light"` 或 `theme="auto"`。

```ts
import express from "express";
import { hatchaRouter } from "@mondaycom/hatcha-server/express";
const app = express();
app.use(express.json());
app.use("/api/hatcha", hatchaRouter({ secret: process.env.HATCHA_SECRET! }));
app.listen(3000);
```

| 包 | 描述 |
|---|---|
| `@mondaycom/hatcha-core` | 挑战生成与加密验证 |
| `@mondaycom/hatcha-react` | React 组件、Provider 和样式 |
| `@mondaycom/hatcha-server` | Next.js 和 Express 服务端处理程序 |

```
git clone https://github.com/mondaycom/HATCHA.git
cd HATCHA
pnpm install
pnpm build
cd examples/nextjs-app
pnpm dev
```

欢迎贡献!有关设置说明和指南,请参阅 CONTRIBUTING.md。
