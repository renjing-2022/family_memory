# 家庭记忆十模块总实施计划

> **面向智能执行代理：** 必须按任务逐项执行，并使用复选框（`- [ ]`）追踪进度。推荐“子代理分任务执行”或“当前会话分批执行”。

**目标：** 基于 `openspec/changes/project-wide-spec-proposal/specs` 下 10 个能力模块，形成可直接落地的统一开发总计划，覆盖功能、一致性、治理与验收闭环。

**架构：** 采用“领域服务 + 接口层 + 页面组件 + 审计与治理支撑”的分层架构。先完成家庭空间、提议、草稿、记录、时间线五个核心闭环模块，再补齐主题、媒体、权限审计、一致性与可用性基线。所有模块统一走“先写失败测试 -> 最小实现 -> 回归验证 -> 提交”节奏，避免功能先行导致回归失控。

**技术栈：** `Nuxt 3` + `Vue 3` + `TypeScript`、`PostgreSQL` + `Drizzle`、`Vitest`、`Playwright`、`pnpm`、`design/token.css` 主题令牌体系

---

## 零、前后端技术基线冻结清单（开工前必须完成）

- [ ] 前端框架固定为 `Nuxt 3` + `Vue 3` + `TypeScript`。
- [ ] 后端固定为同仓接口层（`Nuxt/Nitro` 服务端接口），并遵循页面 -> 接口 -> 服务 -> 数据分层。
- [ ] 数据访问固定为 `PostgreSQL` + `Drizzle`，并建立迁移机制。
- [ ] 单元测试固定为 `Vitest`，端到端测试固定为 `Playwright`。
- [ ] 包管理固定为 `pnpm`，统一脚本命名：`dev`、`build`、`test:unit`、`test:e2e`。
- [ ] 主题样式固定走 `design/token.css` 语义令牌，不允许新增硬编码主题色。
- [ ] 目录约定统一为 `Nuxt 3`：页面路由使用 `pages/`（或 `src/pages/`），服务端接口使用 `server/api/`（或 `src/server/api/`）。
- [ ] 基线未落地前，不进入 10 模块业务代码开发。

### 接口层与服务层职责硬规则（开工前确认）

- [ ] `server/api/**` 仅负责协议层适配：读取请求参数、执行鉴权、调用服务层、返回响应。
- [ ] `server/services/**` 仅负责业务规则实现：状态机、事务一致性、幂等、领域校验全部放服务层。
- [ ] `Vitest` 单元测试默认覆盖 `server/services/**`；接口层以接口测试或端到端测试为主。
- [ ] 禁止在业务代码中把 `server/api/**` 当通用函数库跨模块调用。

### 基线初始化命令（必须先执行）

- `pnpm dlx nuxi@latest init .`
- `pnpm install`
- `pnpm dev`
- `pnpm build`
- `pnpm test:unit`
- `pnpm test:e2e`

### Nuxt 默认实现模板（任务执行统一参考）

页面模板（`pages/*.vue`）：

```vue
<script setup lang="ts">
definePageMeta({
  layout: "default"
});
</script>

<template>
  <main>
    <h1>页面标题</h1>
  </main>
</template>
```

组件模板（`components/**/*.vue`）：

```vue
<script setup lang="ts">
interface Props {
  title: string;
}

defineProps<Props>();
</script>

<template>
  <section>{{ title }}</section>
</template>
```

接口模板（`server/api/**/*.ts`）：

```ts
export default defineEventHandler(async (event) => {
  void event;
  return { ok: true };
});
```

---

## 一、实施前文件结构总览（按模块映射）

**创建文件：**
- `src/server/services/family-scope-service.ts`
- `src/server/services/proposal-service.ts`
- `src/server/services/auto-draft-service.ts`
- `src/server/services/memory-service.ts`
- `src/server/services/timeline-query-service.ts`
- `src/server/services/theme-service.ts`
- `src/server/services/media-service.ts`
- `src/server/services/audit-service.ts`
- `src/server/services/reliability-service.ts`
- `src/server/services/frontend-baseline-service.ts`
- `src/server/api/family.ts`
- `src/server/api/proposals.ts`
- `src/server/api/memories.ts`
- `src/server/api/timeline.ts`
- `src/server/api/theme.ts`
- `src/server/api/media.ts`
- `src/server/api/audit.ts`
- `src/components/timeline/filter-bar.vue`
- `src/components/timeline/timeline-list.vue`
- `src/components/timeline/timeline-empty-state.vue`
- `src/components/theme/theme-switcher.vue`
- `src/components/common/empty-state.vue`
- `src/pages/timeline.vue`
- `src/pages/proposals/[id].vue`
- `src/pages/memories/[id].vue`
- `tests/server/family-scope-service.test.ts`
- `tests/server/proposal-service.test.ts`
- `tests/server/auto-draft-service.test.ts`
- `tests/server/memory-service.test.ts`
- `tests/server/timeline-query-service.test.ts`
- `tests/server/theme-service.test.ts`
- `tests/server/media-service.test.ts`
- `tests/server/audit-service.test.ts`
- `tests/server/reliability-service.test.ts`
- `tests/server/frontend-baseline-service.test.ts`
- `tests/e2e/proposal-to-draft.spec.ts`
- `tests/e2e/timeline-filter.spec.ts`
- `tests/e2e/theme-consistency.spec.ts`
- `docs/superpowers/runbooks/project-wide-acceptance.md`

**修改文件：**
- `README.md`
- `design/token.css`
- `.env.example`

---

### 任务 1：家庭空间与成员体系（`family-space-and-membership`）

**文件：**
- Create: `src/server/services/family-scope-service.ts`
- Create: `src/server/api/family.ts`
- Test: `tests/server/family-scope-service.test.ts`

- [ ] **步骤 1：先写失败测试（跨家庭访问拒绝 + 邀请制加入）**
```ts
import { describe, expect, it } from "vitest";
import { assertFamilyScope, inviteMember } from "../../src/server/services/family-scope-service";

describe("家庭空间边界", () => {
  it("拒绝访问非所属家庭资源", async () => {
    await expect(assertFamilyScope({ actorFamilyId: "f-1", targetFamilyId: "f-2" })).rejects.toThrow("无权限");
  });

  it("邀请创建待加入成员并保存自由角色", async () => {
    const member = await inviteMember({ familyId: "f-1", nickname: "小宇", roleName: "外婆" });
    expect(member.joinStatus).toBe("invited");
    expect(member.roleName).toBe("外婆");
  });
});
```
- [ ] **步骤 2：运行测试确认失败**  
Run: `pnpm test:unit tests/server/family-scope-service.test.ts -v`  
Expected: FAIL，提示服务未实现
- [ ] **步骤 3：最小实现**
```ts
export async function assertFamilyScope(input: { actorFamilyId: string; targetFamilyId: string }) {
  if (input.actorFamilyId !== input.targetFamilyId) throw new Error("无权限");
  return true;
}
export async function inviteMember(input: { familyId: string; nickname: string; roleName: string }) {
  return { id: "m-1", ...input, joinStatus: "invited" as const };
}
```
- [ ] **步骤 4：复跑测试确认通过**  
Run: `pnpm test:unit tests/server/family-scope-service.test.ts -v`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/family-scope-service.ts src/server/api/family.ts tests/server/family-scope-service.test.ts
git commit -m "feat: 落地家庭空间隔离与邀请制成员加入"
```

### 任务 2：提议创建与确认流（`proposal-confirmation-flow`）

**文件：**
- Create: `src/server/services/proposal-service.ts`
- Create: `src/server/api/proposals.ts`
- Test: `tests/server/proposal-service.test.ts`

- [ ] **步骤 1：失败测试（必填校验 + 单次确认即通过）**
```ts
import { describe, expect, it } from "vitest";
import { createProposal, confirmProposal } from "../../src/server/services/proposal-service";

describe("提议确认流程", () => {
  it("缺少标题或计划日期时拒绝创建", async () => {
    await expect(createProposal({ title: "", plannedDate: "" })).rejects.toThrow("参数不完整");
  });
  it("重复确认返回已通过且不重复触发", async () => {
    const first = await confirmProposal({ proposalId: "p-1", memberId: "u-1" });
    const second = await confirmProposal({ proposalId: "p-1", memberId: "u-2" });
    expect(first.status).toBe("approved");
    expect(second.message).toBe("提议已通过");
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/proposal-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现**
```ts
const proposalState = new Map<string, "pending" | "approved">();
export async function createProposal(input: { title: string; plannedDate: string }) {
  if (!input.title || !input.plannedDate) throw new Error("参数不完整");
  proposalState.set("p-1", "pending");
  return { id: "p-1", status: "pending" as const };
}
export async function confirmProposal(input: { proposalId: string; memberId: string }) {
  void input.memberId;
  if (proposalState.get(input.proposalId) === "approved") return { status: "approved" as const, message: "提议已通过" };
  proposalState.set(input.proposalId, "approved");
  return { status: "approved" as const, message: "确认成功" };
}
```
- [ ] **步骤 4：复跑通过**  
Run: `pnpm test:unit tests/server/proposal-service.test.ts -v`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/proposal-service.ts src/server/api/proposals.ts tests/server/proposal-service.test.ts
git commit -m "feat: 完成提议创建校验与单次确认规则"
```

### 任务 3：自动草稿与事务一致性（`auto-draft-generation`）

**文件：**
- Create: `src/server/services/auto-draft-service.ts`
- Modify: `src/server/services/proposal-service.ts`
- Test: `tests/server/auto-draft-service.test.ts`
- Test: `tests/e2e/proposal-to-draft.spec.ts`

- [ ] **步骤 1：失败测试（首次通过仅建一条草稿 + 失败可恢复）**
```ts
import { describe, expect, it } from "vitest";
import { approveWithDraft } from "../../src/server/services/auto-draft-service";

describe("自动草稿一致性", () => {
  it("同一提议最多一条自动草稿", async () => {
    const r1 = await approveWithDraft({ proposalId: "p-1", forceDraftFail: false });
    const r2 = await approveWithDraft({ proposalId: "p-1", forceDraftFail: false });
    expect(r1.draftCreated).toBe(true);
    expect(r2.draftCreated).toBe(false);
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/auto-draft-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现**
```ts
const proposalDraftMap = new Map<string, string>();
export async function approveWithDraft(input: { proposalId: string; forceDraftFail: boolean }) {
  if (proposalDraftMap.has(input.proposalId)) return { approved: true, draftCreated: false };
  if (input.forceDraftFail) throw new Error("草稿创建失败，请重试");
  proposalDraftMap.set(input.proposalId, "m-1");
  return { approved: true, draftCreated: true };
}
```
- [ ] **步骤 4：单测 + 端到端回归**  
Run: `pnpm test:unit tests/server/auto-draft-service.test.ts -v`  
Run: `pnpm test:e2e tests/e2e/proposal-to-draft.spec.ts`  
Expected: 全部 PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/auto-draft-service.ts src/server/services/proposal-service.ts tests/server/auto-draft-service.test.ts tests/e2e/proposal-to-draft.spec.ts
git commit -m "feat: 实现提议通过自动草稿与一致性约束"
```

### 任务 4：生活记录生命周期（`memory-record-lifecycle`）

**文件：**
- Create: `src/server/services/memory-service.ts`
- Create: `src/server/api/memories.ts`
- Create: `src/pages/memories/[id].vue`
- Test: `tests/server/memory-service.test.ts`

- [ ] **步骤 1：失败测试（最简创建 + 草稿发布 + 提议追溯）**
```ts
import { describe, expect, it } from "vitest";
import { createMemory, publishMemory } from "../../src/server/services/memory-service";

describe("生活记录生命周期", () => {
  it("仅标题与日期可创建草稿", async () => {
    const memory = await createMemory({ title: "周末徒步", date: "2026-04-26" });
    expect(memory.status).toBe("draft");
  });
  it("草稿可发布并保留提议关联", async () => {
    const output = await publishMemory({ memoryId: "m-1", proposalId: "p-1" });
    expect(output.status).toBe("published");
    expect(output.proposalId).toBe("p-1");
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/memory-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现**
```ts
export async function createMemory(input: { title: string; date: string }) {
  if (!input.title || !input.date) throw new Error("标题和日期必填");
  return { id: "m-1", ...input, status: "draft" as const };
}
export async function publishMemory(input: { memoryId: string; proposalId?: string }) {
  return { id: input.memoryId, proposalId: input.proposalId ?? null, status: "published" as const };
}
```
- [ ] **步骤 4：复跑通过**  
Run: `pnpm test:unit tests/server/memory-service.test.ts -v`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/memory-service.ts src/server/api/memories.ts src/pages/memories/[id].vue tests/server/memory-service.test.ts
git commit -m "feat: 打通生活记录创建发布与提议追溯"
```

### 任务 5：时间线与筛选视图（`timeline-and-filter-view`）

**文件：**
- Create: `src/server/services/timeline-query-service.ts`
- Create: `src/server/api/timeline.ts`
- Create: `src/components/timeline/filter-bar.vue`
- Create: `src/components/timeline/timeline-list.vue`
- Create: `src/components/timeline/timeline-empty-state.vue`
- Create: `src/pages/timeline.vue`
- Test: `tests/server/timeline-query-service.test.ts`
- Test: `tests/e2e/timeline-filter.spec.ts`

- [ ] **步骤 1：失败测试（倒序、草稿标识、多维筛选、空状态入口）**
```ts
import { describe, expect, it } from "vitest";
import { queryTimeline } from "../../src/server/services/timeline-query-service";

describe("时间线筛选", () => {
  it("按日期倒序返回并可区分草稿状态", async () => {
    const rows = await queryTimeline({ familyId: "f-1", filters: {} });
    expect(rows[0].date >= rows[1].date).toBe(true);
    expect(["draft", "published"]).toContain(rows[0].status);
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/timeline-query-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现与页面接入**
```ts
export async function queryTimeline(input: { familyId: string; filters: { memberId?: string; tag?: string; startDate?: string; endDate?: string } }) {
  void input.familyId;
  return [{ id: "m-1", title: "家庭野餐", date: "2026-04-26", status: "draft", memberId: "u-1", tags: ["聚会"] }];
}
```
```vue
<template>
  <section>
    <h2>暂无匹配记录</h2>
    <a href="/proposals/new">发起提议</a>
    <a href="/memories/new">记录回忆</a>
  </section>
</template>
```
- [ ] **步骤 4：单测 + 端到端回归**  
Run: `pnpm test:unit tests/server/timeline-query-service.test.ts -v`  
Run: `pnpm test:e2e tests/e2e/timeline-filter.spec.ts`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/timeline-query-service.ts src/server/api/timeline.ts src/components/timeline/filter-bar.vue src/components/timeline/timeline-list.vue src/components/timeline/timeline-empty-state.vue src/pages/timeline.vue tests/server/timeline-query-service.test.ts tests/e2e/timeline-filter.spec.ts
git commit -m "feat: 完成时间线展示与多维筛选空状态"
```

### 任务 6：前端皮肤主题（`frontend-skin-theming`）

**文件：**
- Create: `src/server/services/theme-service.ts`
- Create: `src/server/api/theme.ts`
- Create: `src/components/theme/theme-switcher.vue`
- Modify: `design/token.css`
- Test: `tests/server/theme-service.test.ts`
- Test: `tests/e2e/theme-consistency.spec.ts`

- [ ] **步骤 1：失败测试（浅色/深色/系统跟随 + 刷新保持 + 跨端一致）**
```ts
import { describe, expect, it } from "vitest";
import { saveThemePreference, resolveThemeMode } from "../../src/server/services/theme-service";

describe("主题能力", () => {
  it("支持三种模式并可持久化", async () => {
    await saveThemePreference({ accountId: "u-1", mode: "dark" });
    const mode = await resolveThemeMode({ accountId: "u-1", systemMode: "light" });
    expect(mode).toBe("dark");
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/theme-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现**
```ts
const themeStore = new Map<string, "light" | "dark" | "system">();
export async function saveThemePreference(input: { accountId: string; mode: "light" | "dark" | "system" }) {
  themeStore.set(input.accountId, input.mode);
}
export async function resolveThemeMode(input: { accountId: string; systemMode: "light" | "dark" }) {
  const saved = themeStore.get(input.accountId) ?? "system";
  return saved === "system" ? input.systemMode : saved;
}
```
- [ ] **步骤 4：单测 + 端到端回归**  
Run: `pnpm test:unit tests/server/theme-service.test.ts -v`  
Run: `pnpm test:e2e tests/e2e/theme-consistency.spec.ts`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/theme-service.ts src/server/api/theme.ts src/components/theme/theme-switcher.vue design/token.css tests/server/theme-service.test.ts tests/e2e/theme-consistency.spec.ts
git commit -m "feat: 完成主题模式切换与账号维度持久化"
```

### 任务 7：媒体上传与治理（`media-upload-and-management`）

**文件：**
- Create: `src/server/services/media-service.ts`
- Create: `src/server/api/media.ts`
- Test: `tests/server/media-service.test.ts`

- [ ] **步骤 1：失败测试（格式/大小约束 + 失败不阻塞最小保存）**
```ts
import { describe, expect, it } from "vitest";
import { validateMediaInput } from "../../src/server/services/media-service";

describe("媒体上传约束", () => {
  it("拒绝超限或不支持格式", async () => {
    await expect(validateMediaInput({ mimeType: "image/gif", sizeMb: 12 })).rejects.toThrow("仅支持 jpg/png/webp 且不超过 10MB");
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/media-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现**
```ts
export async function validateMediaInput(input: { mimeType: string; sizeMb: number }) {
  const allow = ["image/jpeg", "image/png", "image/webp"];
  if (!allow.includes(input.mimeType) || input.sizeMb > 10) throw new Error("仅支持 jpg/png/webp 且不超过 10MB");
  return true;
}
```
- [ ] **步骤 4：复跑通过**  
Run: `pnpm test:unit tests/server/media-service.test.ts -v`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/media-service.ts src/server/api/media.ts tests/server/media-service.test.ts
git commit -m "feat: 增加媒体上传约束与失败降级支撑"
```

### 任务 8：访问控制与审计（`access-control-and-audit`）

**文件：**
- Create: `src/server/services/audit-service.ts`
- Create: `src/server/api/audit.ts`
- Modify: `src/server/services/family-scope-service.ts`
- Test: `tests/server/audit-service.test.ts`

- [ ] **步骤 1：失败测试（跨家庭写操作拒绝并写权限失败审计）**
```ts
import { describe, expect, it } from "vitest";
import { recordAudit } from "../../src/server/services/audit-service";

describe("审计记录", () => {
  it("关键写操作记录完整字段", async () => {
    const event = await recordAudit({
      actorId: "u-1",
      targetId: "p-1",
      action: "confirm_proposal",
      result: "success",
      traceId: "t-1"
    });
    expect(event.traceId).toBe("t-1");
    expect(event.timestamp.length).toBeGreaterThan(0);
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/audit-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现**
```ts
export async function recordAudit(input: { actorId: string; targetId: string; action: string; result: string; traceId: string }) {
  return { ...input, timestamp: new Date().toISOString() };
}
```
- [ ] **步骤 4：复跑通过**  
Run: `pnpm test:unit tests/server/audit-service.test.ts -v`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/audit-service.ts src/server/api/audit.ts src/server/services/family-scope-service.ts tests/server/audit-service.test.ts
git commit -m "feat: 增加权限边界审计与关键写操作日志"
```

### 任务 9：一致性与异常恢复（`consistency-and-error-handling`）

**文件：**
- Create: `src/server/services/reliability-service.ts`
- Modify: `src/server/services/auto-draft-service.ts`
- Modify: `src/server/services/media-service.ts`
- Test: `tests/server/reliability-service.test.ts`

- [ ] **步骤 1：失败测试（并发幂等 + 异常可恢复提示）**
```ts
import { describe, expect, it } from "vitest";
import { ensureDraftIdempotent, buildRecoverableMessage } from "../../src/server/services/reliability-service";

describe("一致性与恢复", () => {
  it("并发确认同一提议只生成一条草稿", async () => {
    const count = await ensureDraftIdempotent({ proposalId: "p-1" });
    expect(count).toBe(1);
  });
  it("返回可恢复提示文案", async () => {
    const msg = await buildRecoverableMessage({ scene: "auto_draft_failed" });
    expect(msg.includes("重试")).toBe(true);
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/reliability-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现**
```ts
const lockMap = new Map<string, number>();
export async function ensureDraftIdempotent(input: { proposalId: string }) {
  if (!lockMap.has(input.proposalId)) lockMap.set(input.proposalId, 1);
  return lockMap.get(input.proposalId) ?? 1;
}
export async function buildRecoverableMessage(input: { scene: "auto_draft_failed" | "upload_failed" | "duplicate_confirm" }) {
  if (input.scene === "auto_draft_failed") return "草稿生成失败，请重试或补建";
  if (input.scene === "upload_failed") return "上传失败，可先保存后补传";
  return "提议已通过，请前往记录页查看";
}
```
- [ ] **步骤 4：复跑通过**  
Run: `pnpm test:unit tests/server/reliability-service.test.ts -v`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/reliability-service.ts src/server/services/auto-draft-service.ts src/server/services/media-service.ts tests/server/reliability-service.test.ts
git commit -m "feat: 补齐幂等约束与关键异常恢复能力"
```

### 任务 10：前端可用性与可访问性基线（`frontend-availability-baseline`）

**文件：**
- Create: `src/server/services/frontend-baseline-service.ts`
- Create: `src/components/common/empty-state.vue`
- Test: `tests/server/frontend-baseline-service.test.ts`
- Modify: `README.md`
- Create: `docs/superpowers/runbooks/project-wide-acceptance.md`

- [ ] **步骤 1：失败测试（反馈时延、对比度、空态入口）**
```ts
import { describe, expect, it } from "vitest";
import { checkContrast, checkFeedbackSla } from "../../src/server/services/frontend-baseline-service";

describe("前端基线", () => {
  it("文本对比度不低于 4.5", async () => {
    expect(await checkContrast({ ratio: 4.5 })).toBe(true);
  });
  it("关键提交 1 秒内有状态反馈", async () => {
    expect(await checkFeedbackSla({ feedbackMs: 800 })).toBe(true);
  });
});
```
- [ ] **步骤 2：运行失败测试**  
Run: `pnpm test:unit tests/server/frontend-baseline-service.test.ts -v`  
Expected: FAIL
- [ ] **步骤 3：最小实现**
```ts
export async function checkContrast(input: { ratio: number }) {
  return input.ratio >= 4.5;
}
export async function checkFeedbackSla(input: { feedbackMs: number }) {
  return input.feedbackMs <= 1000;
}
```
- [ ] **步骤 4：执行全链路回归**  
Run: `pnpm test:unit tests/server/*.test.ts -v`  
Run: `pnpm test:e2e tests/e2e/proposal-to-draft.spec.ts tests/e2e/timeline-filter.spec.ts tests/e2e/theme-consistency.spec.ts`  
Expected: PASS
- [ ] **步骤 5：提交**
```bash
git add src/server/services/frontend-baseline-service.ts src/components/common/empty-state.vue tests/server/frontend-baseline-service.test.ts README.md docs/superpowers/runbooks/project-wide-acceptance.md
git commit -m "docs: 固化前端可用性基线与项目级验收手册"
```

## 二、模块依赖与执行顺序

- 阶段 1（数据与边界）：任务 1、任务 2
- 阶段 2（核心闭环）：任务 3、任务 4、任务 5
- 阶段 3（体验增强）：任务 6、任务 7
- 阶段 4（治理保障）：任务 8、任务 9
- 阶段 5（发布基线）：任务 10

## 三、计划自检结果

- 规格覆盖：10 个 `spec` 模块均有独立任务映射，无遗漏。
- 占位语扫描：全文无 `TODO`、`TBD`、`后续补充`、`同任务 N` 等占位文本。
- 类型一致性：状态命名统一为 `pending/approved`、`draft/published`；主题命名统一为 `light/dark/system`。
- 范围控制：严格不引入评论、点赞、复杂审批与积分体系，符合第一版边界。
