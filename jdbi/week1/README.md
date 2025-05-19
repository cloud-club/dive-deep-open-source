# week2

---

1. **1주차**: 이슈 #2749 분석 및 JDBI 개발 환경 설정
    - 이슈 상세 분석 및 관련 컴포넌트 파악
    - 개발 환경 설정 (Java, Maven, IDE 설정) ✅
    - 프로젝트 클론 및 빌드해보기 ✅
    - 기본 JDBI 기능 테스트 코드 실행해보기
2. **2주차**: JDBI 아키텍처 및 핵심 모듈 분석
    - 핵심 클래스 구조 파악
    - SQL Object 인터페이스와 Fluent API 이해하기
    - 관련 테스트 코드 분석 및 실행
    - 간단한 예제 구현해보기

---

### #2749 이슈 내용

https://github.com/jdbi/jdbi/issues/2749

The SQL Object interface supports simple configuration of how many rows to process at a time via the `@BatchChunkSize` annotation. I think it would be beneficial to support a batch size argument for `PreparedBatch` directly, so that the user could feed in any number of bind parameter sets, but when `execute` is called, multiple calls would be made based on the chunk size.

Currently, we have to do this manually when using `PreparedBatch` outside of the SQL Object interface, which tends to result in boilerplate code that looks something like this:

```java
int counter = 0;
try(PreparedBatch insertQuery = getHandle().prepareBatch("insert :value into some_table")
{
   for (Integer values : valueList)
   {
      insertQuery.bind("value", value).add();
      counter++;
      if (counter >= 10000)
      {
         insertQuery.execute();
         counter = 0;
      }
   }
   if (counter > 0)
   {
      insertQuery.execute();
  }
}
```

If `PreparedBatch` supported a chunk size, we could make this much more concise:

```java
try(PreparedBatch insertQuery = getHandle().prepareBatch("insert :value into some_table")
{
   insertQuery.setBatchChunkSize(10000);
   values.forEach(value -> insertQuery.bind("value", value).add());
   insertQuery.execute();
}
```

### 요약

> 기존에 SQLObject 인터페이스에서는 @BatchChunkSize 어노테이션을 제공. 한 번에 배치에 몇 개의 파라미터를 바인딩 할 수 있는지 결정할 수 있는 데에 비해, PreparedBatch 객체를 사용할 때에는 이와 같은 기능이 없음. 이는 클라이언트 측에 불필요한 보일러플레이트 코드의 작성을 강요함 → 추가해줘
> 

## 핵심 클래스 및 패키지

1. **org.jdbi.v3.core.statement.PreparedBatch**
    - 주요 타겟 클래스로, 여기에 청크 크기 설정을 위한 새 메서드를 추가해야 합니다.
    - 이 클래스는 SQL 문을 여러 번 실행하기 위한 배치 처리를 담당합니다.
    - `SqlStatement` 클래스를 상속받고 `ResultBearing` 인터페이스를 구현합니다.
2. **org.jdbi.v3.sqlobject.statement.BatchChunkSize**
    - 현재 SQL 객체에서 배치 청크 크기를 지정하기 위해 사용되는 어노테이션입니다.
    - 이 어노테이션의 처리 로직을 참고하여 `PreparedBatch`에 유사한 기능을 구현할 수 있습니다.
3. **org.jdbi.v3.sqlobject.statement.internal.SqlBatchHandler**
    - SQL 객체에서 배치 처리와 청크 크기를 어떻게 구현하는지 이해하는 데 도움이 됩니다.
    - 이 클래스는 `@BatchChunkSize` 어노테이션을 실제로 처리하는 로직을 포함합니다.
4. **org.jdbi.v3.core.statement.SqlStatement**
    - `PreparedBatch`의 부모 클래스로, 모든 SQL 문 처리의 기본 기능을 제공합니다.
    - 새로운 메서드가 기존 클래스 계층 구조와 일관되게 동작하도록 이 클래스를 이해해야 합니다.

### 예상되는 문제

1. 처음엔 단순하게 execute를 감싸서 그대로 넣으면 될 줄 알았는데, setChinkSize, bind 호출 시점과 다르게, 청크를 나누는건 execute 호출 시점에 일어나야함
2. 진짜 문제는 현재는 execute 호출시점에 이전에 바인드한 모든 파람을 날려버림
3. Batch 호출 시, 중간 요청이 실패하면 어떻게 처리할지 결정 →기존 SQL 객체 인터페이스의 `@SqlBatch` 어노테이션에서는 `transactional` 속성을 통해 트랜잭션 처리를 제어할 수 있음. 새 구현도 이와 일관되게 동작해야함.
    
    → 전체 배치 롤백? 아니면 성공한 청크는 커밋?
    

⇒ 오랜만에 보려니 다 까먹어서.. SQLObject @BatchChunkSize 어떻게 동작하는지부터 다시 한 번 봐야될 것 같습니다.

![스크린샷 2025-05-19 오후 9.52.44.png](week2%201f64b3917fae8074ba9eea9b4f341927/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-05-19_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_9.52.44.png)

테스트 돌려보는데 3분걸리네요..

### 주요코드

**PreparedBatch**

```java
public class PreparedBatch extends SqlStatement<PreparedBatch> implements ResultBearing {
    // 바인딩 저장
    private final List<PreparedBinding> bindings = new ArrayList<>();
    final Map<PrepareKey, Function<String, Optional<Function<Object, Argument>>>> preparedFinders = new HashMap<>();
        ...
```

```java
    /**
     * Add the current binding as a saved batch and clear the binding.
     * @return this
     */
    public PreparedBatch add() {
        final PreparedBinding currentBinding = getBinding();
        if (currentBinding.isEmpty()) {
            throw new IllegalStateException("Attempt to add() an empty batch, you probably didn't mean to do this "
                + "- call add() *after* setting batch parameters");
        }
        bindings.add(currentBinding);
        getContext().setBinding(new PreparedBinding(getContext()));
        return this;
    }
    
        /**
     * Bind arguments positionally, add the binding as a saved batch, and then clear the current binding.
     * @param args the positional arguments to bind
     * @return this
     */
    public PreparedBatch add(Object... args) {
        for (int i = 0; i < args.length; i++) {
            bind(i, args[i]);
        }
        add();
        return this;
    }

    /**
     * Bind arguments from a Map, add the binding as a saved batch,
     * then clear the current binding.
     *
     * @param args map to bind arguments from for named parameters on the statement
     * @return this
     */
    public PreparedBatch add(Map<String, ?> args) {
        bindMap(args);
        add();
        return this;
    }
```

- 현재 바인딩을 배치에 추가하고 바인딩을 초기화

```java

    /**
     * Execute the batch and return the number of rows affected for each batch part.
     * Note that some database drivers might return special values like {@link Statement#SUCCESS_NO_INFO}
     * or {@link Statement#EXECUTE_FAILED}.
     *
     * @return the number of rows affected per batch part
     * @see Statement#executeBatch()
     */
    public int[] execute() {
        try {
            return internalBatchExecute().updateCounts;
        } finally {
            close();
        }
    }
    
    private ExecutedBatch internalBatchExecute() {
        if (!getBinding().isEmpty()) {
            add();
        }

        beforeTemplating();

        final StatementContext ctx = getContext();

        ParsedSql parsedSql = parseSql();
        String sql = parsedSql.getSql();
        ParsedParameters parsedParameters = parsedSql.getParameters();

        try {
            try {
                stmt = createStatement(sql);

                getContext().addCleanable(() -> cleanupStatement(stmt));
                getConfig(SqlStatements.class).customize(stmt);
            } catch (SQLException e) {
                throw new UnableToCreateStatementException(e, ctx);
            }

            if (bindings.isEmpty()) {
                return new ExecutedBatch(stmt, new int[0]);
            }

            beforeBinding();

            try {
                ArgumentBinder binder = new ArgumentBinder.Prepared(this, parsedParameters, bindings.get(0));
                for (Binding binding : bindings) {
                    ctx.setBinding(binding);
                    binder.bind(binding);
                    stmt.addBatch();
                }
            } catch (SQLException e) {
                throw new UnableToExecuteStatementException("Exception while binding parameters", e, ctx);
            }

            beforeExecution();

            try {
                final int[] modifiedRows = SqlLoggerUtil.wrap(stmt::executeBatch, ctx, getConfig(SqlStatements.class).getSqlLogger());

                afterExecution();

                ctx.setBinding(new PreparedBinding(ctx));

                return new ExecutedBatch(stmt, modifiedRows);
            } catch (SQLException e) {
                throw new UnableToExecuteStatementException(Batch.mungeBatchException(e), ctx);
            }
        } finally {
            bindings.clear();
        }
    }
```

- 배치를 실행하고 각 배치 부분별 영향받은 행 수 반환함
- 내부적으로 for문 돌면서 그동안 bind한 바인딩(아규먼트)들 추가하고 한 번에 실행시킴
- `UnableToExecuteStatementException` 던지면 될듯?

**SqlStatement**

```java
    /**
     * Bind an argument positionally
     *
     * @param position position to bind the parameter at, starting at 0
     * @param value    to bind
     *
     * @return the same Query instance
     */
    public final This bind(int position, Object value) {
        getBinding().addPositional(position, value);
        return typedThis;
    }
```