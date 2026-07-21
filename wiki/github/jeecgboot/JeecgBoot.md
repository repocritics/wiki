# jeecgboot/JeecgBoot

An enterprise Java low-code platform that pairs code generation with a built-in AI application layer, and adds natural-language "Skills" to generate systems, forms, flows, reports, and dashboards.

## What it is

JeecgBoot is a low-code development platform for enterprise J2EE applications that supports both low-code and zero-code modes. In zero-code mode, business users assemble systems through online configuration; in low-code mode, the platform generates front-end and back-end code, table-creation SQL, and menu permissions that run as generated. It targets Java teams building MIS, OA, ERP, CRM, SaaS, and AI knowledge-base systems who want to cut repetitive work while retaining the ability to hand-merge and extend generated code. It also ships an AI application platform and "AI Skills" for generating artifacts from natural language.

## Key features

- Dual low-code plus zero-code development: online forms, form/flow/portal designers, and a code generator (jeecg-codegen) with multiple templates for single-table, tree, and one-to-many models.
- AI Skills for natural-language generation of complete systems, forms, flowcharts, reports, and dashboards.
- A built-in AI application platform: chat assistant, RAG knowledge base, flow orchestration, model management, and MCP plugin configuration, compatible with ChatGPT, DeepSeek, Ollama, Qwen, and Zhipu.
- Reporting and visualization via JimuReport and big-screen/dashboard design via JimuBI.
- Granular RBAC with button-level and row/column-level data permissions, plus multi-tenant SaaS support.
- Switchable monolith or microservice deployment on Spring Cloud Alibaba (Nacos, Gateway, Sentinel, Skywalking), with a companion Uniapp mobile framework for H5, mini-program, Android, iOS, and HarmonyOS.

## Tech stack

- Backend: Java (default JDK 17, also 21/24), Spring Boot 4.1.0, Spring Cloud Alibaba 2025.1.0.0, MyBatis-Plus 3.5.16, Apache Shiro 3.0.0 with JWT 4.5.0, Alibaba Druid 1.2.28, Redis, and Maven.
- Frontend: Vue 3, TypeScript, Vite 6, Ant Design Vue 4, Pinia, ECharts, vxe-table, and qiankun; requires Node 20+ and pnpm 9+.
- Reporting: JimuReport 2.1.5.
- Database support: MySQL (default script provided), Oracle, SQL Server, PostgreSQL, MariaDB, plus the domestic Dameng, Kingbase, and TiDB.
- Current release 3.9.3; Apache-2.0 licensed; AI integration referenced through topics such as langchain4j and spring-ai.

## When to reach for it

- You are building enterprise Java admin systems (MIS, OA, ERP, CRM, or SaaS) and want to eliminate repetitive CRUD and permission work.
- You want a generate-then-hand-merge workflow that keeps flexibility rather than a pure no-code black box.
- You want a low-code platform and an AI application/RAG layer in the same product.
- You need domestic-stack compatibility (Kylin OS, Dameng, Kingbase, TongWeb) for procurement requirements.

## When *not* to reach for it

- Your stack is not Java/Spring; the platform is built around Spring Boot and MyBatis-Plus.
- You want a fully hosted no-code SaaS without self-hosting and operating the platform.
- Your application is small or simple, where a full multi-module low-code platform is heavier than needed.
- Your team needs English-first documentation and community; the primary README, docs, and QQ community are Chinese.

## Maturity signal

JeecgBoot has been developed since 2018, is at version 3.9.3, is Apache-2.0 licensed, and is actively maintained (last push mid-2026). The long lifespan, frequent versioned releases, and a very large fork count relative to stars indicate a mature platform that is heavily reused as a project starting point in practice, not just starred. The disclaimer asking derivatives not to impersonate the official brand further signals a widely forked, commercially significant codebase. Treat it as mature and actively maintained.

## Alternatives

- Dify — use it instead when you only need the AI application and RAG layer and do not need Java code generation; the README itself likens its AI platform to Dify.
- RuoYi — use it instead when you want a lighter Spring Boot admin scaffold without the low-code designers and AI layer.
- Appsmith or Budibase — use them instead when you want an open-source low-code internal-tool builder not tied to the Java/Spring stack.

## Notes

- The README enumerates 44 numbered capabilities, spanning code generation, data permissions, reporting, monitoring, multi-tenancy, and mobile, which is an unusually broad single-platform surface.
- Domestic (信创) compatibility is a first-class concern, covering Kylin-family operating systems, Dameng and Kingbase databases, and TongWeb/TongRDS middleware.
- The platform offers three parallel branches for different security stacks: Spring Boot 4 with Shiro (main), Spring Boot 3 with Spring Authorization Server, and Spring Boot 3 with Sa-Token.

## Tags

java, low-code, spring-boot, vue3, code-generation, llm, rag, mcp, enterprise, mybatis, workflow, ai-agent
