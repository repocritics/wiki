# mybatis/mybatis-3

> A SQL mapper for Java — you write the SQL by hand, MyBatis binds the parameters and maps the result rows. Deliberately not an ORM.

[GitHub repo](https://github.com/mybatis/mybatis-3) ·
[Official website](https://mybatis.org/mybatis-3) ·
[License: Apache-2.0](https://github.com/mybatis/mybatis-3/blob/master/LICENSE)

## Overview

MyBatis is a persistence framework that sits between the "raw JDBC" and "full ORM" ends of the spectrum. You supply the SQL — in XML mapper files or Java annotations — and MyBatis handles the mechanical parts JDBC makes tedious: binding parameters into `PreparedStatement`, iterating the `ResultSet`, mapping columns onto POJO fields, and managing the connection/statement lifecycle. It does *not* generate SQL from an object model, track entity state, or manage a persistence context the way JPA/Hibernate do. That is the entire point.[^1]

The project descends from **iBATIS**, started by Clinton Begin around 2002 and hosted at Apache. The team left Apache in 2010 and rebranded the 3.x line as MyBatis; the GitHub repository was created in 2013 when the code moved off Google Code.[^2] So the "3" in `mybatis-3` is version continuity with iBATIS 3, not a young project — the design is over two decades old and stable to the point of being nearly frozen at the core.

The defining tradeoff: MyBatis gives you full control over the SQL (good for complex queries, legacy schemas, stored procedures, DBA-tuned statements) at the cost of writing and maintaining that SQL yourself. Teams that want the database abstracted away should not use it; teams that consider SQL a first-class artifact usually prefer it to Hibernate. It remains extremely popular in the Java-in-Asia ecosystem, where it is arguably the default data layer.

## Getting Started

Maven coordinate (check Maven Central for the current 3.5.x patch):

```xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>3.5.16</version>
</dependency>
```

A mapper interface bound to SQL by annotation:

```java
public interface UserMapper {
  @Select("SELECT id, name FROM users WHERE id = #{id}")
  User findById(long id);
}
```

Wire up a session factory and run it:

```java
InputStream cfg = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(cfg);

try (SqlSession session = factory.openSession()) {
  UserMapper mapper = session.getMapper(UserMapper.class);
  User u = mapper.findById(1L);
}
```

The equivalent XML mapper (preferred for anything non-trivial, because dynamic SQL lives here):

```xml
<select id="findById" resultType="User">
  SELECT id, name FROM users WHERE id = #{id}
</select>
```

## Architecture / How It Works

The runtime is built around a small set of objects:

- **`SqlSessionFactory`** — built once from `mybatis-config.xml` (or Java config). Immutable, thread-safe, application-scoped.
- **`SqlSession`** — a short-lived, **not thread-safe** unit of work wrapping one JDBC connection. One per request/transaction; always closed.
- **Mappers** — interfaces whose methods map to statements. At runtime MyBatis creates a JDK dynamic proxy per interface, so you never write the JDBC glue.
- **`Executor`** — the layer that actually runs statements. Three modes: `SIMPLE` (a statement per call), `REUSE` (caches `PreparedStatement`), and `BATCH` (queues DML for a single `executeBatch`).

**Parameter substitution has two syntaxes and this is the single most important thing to understand.** `#{param}` becomes a `?` placeholder in a `PreparedStatement` — parameterized and injection-safe. `${param}` performs raw string interpolation into the SQL text before it is compiled — never safe with untrusted input, and the source of most MyBatis SQL-injection CVEs in downstream apps.[^3] `${}` exists for cases where the *structure* must vary (dynamic table/column names, `ORDER BY` direction), which cannot be parameterized.

**Result mapping** is done with `<resultMap>`: it maps columns to fields and supports `<association>` (one-to-one) and `<collection>` (one-to-many) for object graphs. Nested results can be assembled from a single join, or lazily via nested selects — the latter is where N+1 query problems appear.

**Dynamic SQL** is MyBatis's headline feature: `<if>`, `<choose>`, `<where>`, `<set>`, `<foreach>`, `<trim>`, and `<bind>` elements build statements conditionally, with OGNL expressions for the conditions. This is why hand-written SQL stays maintainable — you are not string-concatenating WHERE clauses in Java.

**TypeHandlers** convert between JDBC types and Java types (enums, JSON columns, custom value objects). **Caching** has two levels: a per-`SqlSession` local cache (always on) and an opt-in per-namespace second-level cache enabled with `<cache/>`.

## Production Notes

**Second-level cache is a footgun in clustered deployments.** The built-in `<cache>` is per-JVM and not distributed. In a multi-instance deployment it produces stale reads because one node's writes do not invalidate another node's cache, and it caches by mapper namespace — a write in one namespace does not evict a cached read that touched the same table from another namespace. Most experienced teams leave it off and use an external cache (Redis/EhCache via the `mybatis-redis`/`mybatis-ehcache` integrations) with explicit invalidation, or no MyBatis caching at all.

**N+1 from lazy nested selects.** `<association>`/`<collection>` configured with `select=` issues one query per parent row. It is convenient and quietly catastrophic under load. Prefer join-based result mapping (`<collection>` populated from a single joined query) and reserve nested selects for low-cardinality relations.

**`SqlSession` lifecycle.** A `SqlSession` holds a connection and a first-level cache; keeping one open across requests leaks connections and serves stale cached rows. Outside Spring you manage this manually. With **mybatis-spring** / **mybatis-spring-boot-starter**, `SqlSessionTemplate` makes mappers thread-safe and ties sessions to Spring's transaction scope — this is how the vast majority of production apps run MyBatis, and it removes most lifecycle footguns.

**`${}` audits.** Because `${}` is the injection vector, code review and static analysis should treat every `${}` as suspicious. Grep for it. Legitimate uses (sort direction, whitelisted column names) should validate the value against an allow-list before it reaches the mapper.

**Executor + batch.** Large inserts/updates should use `ExecutorType.BATCH`; the default `SIMPLE` executor round-trips each statement. Mixing batch and non-batch operations in one session, and reading generated keys under batch mode, both have surprising semantics — test them.

**Version stability.** The 3.5.x line has been backward-compatible for years; upgrades are usually painless. The corollary is that the core evolves slowly — do not expect new high-level features, only maintenance, JDK-compatibility fixes, and dependency updates.

## When to Use / When Not

**Use when:**
- Your SQL is complex, hand-tuned, or must run against a schema you do not control.
- You work with stored procedures, reporting queries, or vendor-specific SQL.
- You want the database to remain visible and reviewable, not abstracted behind an entity model.
- You're in an ecosystem (much of the Java/Spring world in Korea, China, Japan) where MyBatis is the team default.

**Avoid when:**
- You want persistence largely automated — entity state tracking, dirty checking, cascade, lazy graph loading. Hibernate/JPA is built for that.
- You want compile-time-checked SQL. String-in-XML SQL fails at runtime; jOOQ or Exposed catch errors at compile time.
- Your domain is a simple CRUD model with no interesting queries — a repository abstraction (Spring Data JPA) will be less boilerplate.

## Alternatives

- hibernate/hibernate-orm — use instead when you want a full JPA ORM that generates SQL and manages entity state, and you're willing to give up direct SQL control.
- jOOQ/jOOQ — use instead when you want type-safe, compile-time-checked SQL as a Java DSL rather than SQL-in-XML.
- spring-projects/spring-data-jpa — use instead when you're all-in on Spring and want repository interfaces over a JPA provider.
- baomidou/mybatis-plus — use on top of MyBatis when you want generated CRUD, pagination, and less mapper boilerplate (dominant in the Chinese ecosystem).
- JetBrains/Exposed — use instead in Kotlin projects wanting an idiomatic SQL DSL and lightweight DAO layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| iBATIS 1.x | ~2002 | Original data-mapper framework at Apache (Clinton Begin). |
| MyBatis 3.0 | 2010 | Rebrand of iBATIS 3 after leaving Apache; annotations, dynamic SQL engine. |
| 3.2 | 2012 | Logging, provider annotations, performance work. |
| 3.4 | 2016 | Multiple result-set support, executor/config refinements. |
| 3.5.0 | 2019 | Java 8 baseline, `Optional` return support, JPMS-aware packaging. |
| 3.5.x | 2019–2026 | Ongoing maintenance; JDK 17–25 build/test matrix, dependency and bug fixes. |

## References

[^1]: MyBatis documentation — Introduction. https://mybatis.org/mybatis-3/
[^2]: MyBatis Wikipedia entry — history of the iBATIS→MyBatis rebrand and move off Apache/Google Code. https://en.wikipedia.org/wiki/MyBatis
[^3]: MyBatis documentation — "String Substitution" and the `#{}` vs `${}` distinction. https://mybatis.org/mybatis-3/sqlmap-xml.html
[^4]: mybatis-spring integration project. https://mybatis.org/spring/

## Tags

java, sql-mapper, orm-alternative, persistence, jdbc, database, spring, xml-mapping, dynamic-sql, data-access
