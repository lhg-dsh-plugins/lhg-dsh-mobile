# 本版修复（BUGFIX）· 0.6.16 → 0.6.17

## Bug：IPv4-mapped 字面量的宽松解析绕过严格 IP 口径，使带前导零的伪字面量被当作合法地址授权

`src/network.ts` 的 `parseIp()` 对 `::ffff:a.b.c.d` 形式的 IPv4-mapped 地址走 `mappedIpv4()` 快捷路径：正则 `\d{1,3}` 允许前导零，随后用宽松的 `parseIpv4()` 转换，整个过程**绕过了** `parseIp()` 其余分支都必经的严格 `isIP()` 门（`src/ip.ts`，与 `net.isIP` 语义一致：严格点分四段、无前导零）。

后果：`::ffff:010.0.0.1`、`::ffff:10.0.0.01` 这类在任何其他写法下都不是合法 IP 的字面量（`isIP()` 返回 0、展开写法 `0:0:0:0:0:ffff:010.0.0.1` 被拒），却经 mapped 快捷路径被当作 `10.0.0.1` 接受，进而：

- `addressAllowed('::ffff:010.0.0.1', [parseCidr('10.0.0.0/8')])` 返回 `true`（直连客户端可信 CIDR 判定被伪字面量命中）；
- `isLoopbackAddress('::ffff:127.0.0.01')` 返回 `true`（loopback 资格判定，关系到 HTTP 明文开发监听资格）；
- `parseCidr('::ffff:010.0.0.0/8')` 被接受为 `10.0.0.0/8`。

同一文件中 `isGloballyRoutableIpv4()` 已用严格八位组解析（`parseStrictIpv4Octets`），mapped 路径是全仓唯一未设严格门的入口。

## 证据链（修复前）

基线（上游 HEAD `26872ce`，v0.6.16）：typecheck ✅、build ✅、全部 verify 检查门 ✅；单元测试 2431 通过 / 35 失败，35 个失败全部为沙箱环境 UDP 广播权限（`discovery_broadcast_EPERM`）与 mDNS 期望差异，集中于 `gateway-event-stream` / `udp-discovery-fixture` 与 gateway 的 Bonjour 两例，与本修复无关（修复后逐项比对，失败集合不变）。

最小复现（vitest，断言期望的安全语义）：

```
tests/repro-network.test.ts（临时文件，验证后已并入 config-network.test.ts 并删除）
  ✓ canonical mapped form stays allowed
  ✓ leading-zero mapped forms are not valid IP literals anywhere else
  × leading-zero mapped forms must not be authorized as the plain address
      addressAllowed('::ffff:010.0.0.1', [10.0.0.0/8]) === true（期望 false）
      addressAllowed('::ffff:10.0.0.01', [10.0.0.0/8]) === true（期望 false）
  × leading-zero mapped loopback must not classify as loopback
      isLoopbackAddress('::ffff:127.0.0.01') === true（期望 false）
→ 2 failed / 2 passed（修复前）
```

## 根因

`mappedIpv4()` 只用正则形状提取点分四段就直接返回，`parseIp()` 拿到 mapped 结果便不再经过 `isIP()` 严格校验；而 `parseIpv4()` 本身按设计是宽松的数值转换（其严格性历来由调用前的 `isIP()` 保证），mapped 快捷路径恰好把这道门短路了。

## 改动文件

- `src/network.ts`（`mappedIpv4`，+5/−1）：提取出的点分四段必须先通过本文件已有的 `parseStrictIpv4Octets()` 严格校验（无前导零、每段 ≤255），否则不认作 mapped 地址，交回 `parseIp()` 的常规严格路径处理（最终抛 `invalid IP address`）。规范形式 `::ffff:10.0.0.1` 行为不变。
- `tests/config-network.test.ts`（+12）：新增回归用例「rejects IPv4-mapped literals whose dotted quad is not a strict IP literal」，覆盖 `addressAllowed`、展开写法对照、`parseCidr` 抛错、`isLoopbackAddress` 四面；并补 `isLoopbackAddress` 的测试导入。
- 版本同步（本版发布动作，非上游改动）：`package.json`、`packages/question-fixes/package.json`、`package-lock.json` 三处版本字段 0.6.16 → 0.6.17（满足仓库自身 `check:version` 的 lock 一致性门）。

## 复验结果（修复后）

- 复现断言全过：临时 repro 文件 4/4 PASS；并入后的 `tests/config-network.test.ts` 45/45 PASS。
- `npm run typecheck` ✅ exit 0。
- 全量 `npm test`：与基线逐项比对——通过数 +1（新增回归用例），失败集合与基线完全一致（仍为沙箱 UDP/mDNS 环境失败 35 个，无新增失败）。
- `npm run build` ✅；`check:version` / `check:docs` / `check:companion-package` / `check:funnel-licenses` / `check:mobile-release` 在成品目录复跑全过。
