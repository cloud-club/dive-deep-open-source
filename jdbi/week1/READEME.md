# 오픈소스 딥 다이브

### **자기 소개**

- 장철희, 졸유하고 DBA 인턴 중
- 관심분야: 백엔드(JVM), 데이터 엔지니어링
- 오픈소스 관심계기: 재밌어보여서..?

## 1. JDBI 프로젝트 개요

[https://github.com/jdbi/jdbi](https://github.com/jdbi/jdbi)

JDBI는 Java에서 데이터베이스에 접근하기 위한 오픈소스 라이브러리로, 기존 JDBC(Java Database Connectivity)의 복잡성을 단순화하고 더 자연스러운 API를 제공

## 2. JDBI 주요 목적 및 기능

JDBI는 JDBC 위에 구축된 편의성 라이브러리로 

- **SQL 중심의 접근 방식**: SQL을 숨기거나 추상화하지 않고, 데이터베이스의 기본 언어로서 SQL을 직접 사용하도록 설계됨. 이를 통해 프로그래머와 데이터 엔지니어가 같은 언어를 사용할 수 있음.
- **자연스러운 Java API**: JDBI는 Java 컬렉션 프레임워크를 쿼리 결과에 사용하고, SQL 문을 외부화하는 편리한 방법을 제공하며, 어떤 데이터베이스에서도 명명된 파라미터를 지원함.
- **두 가지 API 스타일**:
    - 유창한(Fluent) API - 메서드 체이닝을 통한 직관적인 방식
    - SQL 객체(SQL Object) 스타일 - 인터페이스에 애노테이션을 사용하여 SQL 작업 정의
- **경량 설계**: ORM(Object-Relational Mapping)이 아니며, 세션 캐시나 변경 추적 같은 복잡한 기능이 없음. 대신 간단하고 직관적인 데이터베이스 작업에 중점
- JDBI와 JDBC  비교
    
    JDBI와 JDBC는 모두 Java에서 데이터베이스에 접근할 때 사용하는 라이브러리이지만, **JDBI는 JDBC 위에 구축된 추상화 계층**
    
    더 간결하고 생산적인 코드를 작성할 수 있게 해줌.
    
    ## JDBC 예시 코드
    
    ```java
    import java.sql.*;
    
    public class SelectExample {
        static final String DB_URL = "jdbc:mysql://localhost/TUTORIALSPOINT";
        static final String USER = "guest";
        static final String PASS = "guest123";
        static final String QUERY = "SELECT id, first, last, age FROM Employees";
    
        public static void main(String[] args) {
            try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASS);
                 Statement stmt = conn.createStatement();
                 ResultSet rs = stmt.executeQuery(QUERY)) {
    
                while (rs.next()) {
                    System.out.print("ID: " + rs.getInt("id"));
                    System.out.print(", Age: " + rs.getInt("age"));
                    System.out.print(", First: " + rs.getString("first"));
                    System.out.println(", Last: " + rs.getString("last"));
                }
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
    ```
    
    - 연결, 쿼리 실행, 결과 매핑, 자원 해제까지 모든 과정을 직접 처리
    
    ## JDBI 예시 코드
    
    ```java
    import org.jdbi.v3.core.Jdbi;
    
    public class JdbiExample {
        public static void main(String[] args) {
            String jdbcUrl = "jdbc:h2:mem:";
            Jdbi jdbi = Jdbi.create(jdbcUrl);
    
            jdbi.useHandle(handle -> {
                handle.execute("CREATE TABLE employees (id INT, first VARCHAR, last VARCHAR, age INT)");
                handle.execute("INSERT INTO employees VALUES (100, 'Zara', 'Ali', 18)");
                handle.execute("INSERT INTO employees VALUES (101, 'Mahnaz', 'Fatma', 25)");
    
                handle.createQuery("SELECT id, first, last, age FROM employees")
                    .mapToMap()
                    .forEach(row -> System.out.println(row));
            });
        }
    }
    ```
    
    - 연결, 자원 관리, 결과 매핑이 간결, SQL 바인딩이나 매핑도 쉽다
    
    Sql ObjECT API
    
    ```java
    public interface EmployeeDao {
        @SqlUpdate("CREATE TABLE employees (id INT, first VARCHAR, last VARCHAR, age INT)")
        void createTable();
        
        @SqlUpdate("INSERT INTO employees VALUES (:id, :first, :last, :age)")
        void insert(@Bind("id") int id, @Bind("first") String first, 
                    @Bind("last") String last, @Bind("age") int age);
        
        @SqlQuery("SELECT * FROM employees")
        List<Employee> listEmployees();
    }
    
    // 사용 예
    Jdbi jdbi = Jdbi.create(jdbcUrl);
    EmployeeDao dao = jdbi.onDemand(EmployeeDao.class);
    dao.createTable();
    dao.insert(100, "Zara", "Ali", 18);
    List<Employee> employees = dao.listEmployees();
    ```
    
    ## 비교 요약
    
    | 항목 | JDBC | JDBI |
    | --- | --- | --- |
    | 코드 길이 | 길고 반복적임 | 짧고 간결함 |
    | 자원 관리 | 직접 처리해야 함 | 자동으로 처리됨 |
    | 결과 매핑 | 수동으로 객체 변환 필요 | mapTo, mapToBean 등으로 자동 매핑 지원 |
    | 바인딩 방식 | PreparedStatement의 `?`만 지원 | 위치, 이름 기반 바인딩 모두 지원 |
    | 학습 곡선 | 낮음(기본 API), 유지보수 어려움 | 직관적, 유지보수 용이 |
    | 확장성 | 추가 기능 직접 구현 필요 | 플러그인, 확장 API 제공 |

## 3. 대표적인 사용 사례

- 기업 사례가 별로 없음..

**Next Insurance**

글로벌 인슈어테크 기업인 Next Insurance에서는 5년 이상 JDBI를 주요 데이터 액세스 계층으로 사용해왔습니다. 복잡한 ORM 대신 JDBI의 간결함과 명확성을 활용해 대규모 서비스를 운영하고 있습니다

> “SQL을 숨기지 않고 명확하게 사용할 수 있다는 점, 그리고 불필요한 ‘마법’이 없다는 점이 대규모 서비스 운영에 큰 도움이 되었다”
> 

https://youtu.be/jd3brL00j-c?si=Eey5XGCkd57EVbw2

## 4. 프로젝트 선택 이유

제가 JDBI를 선택한 개인적인 이유

- **Java 기반**
- 스프링 같은 대규모 프레임워크에 비해 상대적으로 코드베이스가 작아 기여하기 더 용이(?)해 보임
- 데이터 액세스 레이어에 대한 궁금증 (배치 처리 시 중간에 실패하면 어떻게 처리할까?)도 있어서 이번 기회에 같이 알아보면 좋을듯

## 5. 프로젝트 구조 및 특징 분석

### 폴더/파일 구조 요약

JDBI의 주요 디렉토리와 핵심 파일 구조:

- **core**: 핵심 기능을 담당하는 모듈
- **plugins**: 다양한 플러그인 (Guava, JodaTime, Spring 등)
- **examples**: 사용 예제 모듈

### 핵심 기능 및 컴포넌트

1. **Jdbi 클래스**: 데이터베이스 연결을 관리하는 주요 진입점
2. **Handle**: 데이터베이스에 대한 단일 연결을 나타내며 다양한 작업 실행
3. **Query**: SQL 쿼리를 표현하고 실행
4. **Update**: 데이터 변경 작업 처리
5. **PreparedBatch**: 여러 매개변수 집합으로 단일 문을 배치 처리

### 빌드 및 실행 방법(깃허브에 다..)

- Java 11 이상 필요 (최신 버전은 Java 17 이상 요구)
- Maven을 통한 빌드 관리
- 테스트에는 주로 PostgreSQL 사용한다고 함.

## 6. 향후 6주간 스터디 계획

### 효과적인 학습/분석 방법

- 기본 개념 이해 후 실제 코드로 직접 구현해보는 방식으로 진행함
- 테스트 코드 분석을 통해 의도된 동작 파악하는 데 중점을 둠
    - 테스트 코드가 꽤 많이 작성되어 있어서, 돌려보면서 감잡아가면 될듯함..
- 이슈 #2749에 관련된 코드 흐름 우선적으로 추적하는 방식으로 접근
    
    https://github.com/jdbi/jdbi/issues/2749
    

### 주차별 계획

1. **1주차**: 이슈 #2749 분석 및 JDBI 개발 환경 설정
    - 이슈 상세 분석 및 관련 컴포넌트 파악
    - 개발 환경 설정 (Java, Maven, IDE 설정)
    - 프로젝트 클론 및 빌드해보기
    - 기본 JDBI 기능 테스트 코드 실행해보기
2. **2주차**: JDBI 아키텍처 및 핵심 모듈 분석
    - 핵심 클래스 구조 파악
    - SQL Object 인터페이스와 Fluent API 이해하기
    - 관련 테스트 코드 분석 및 실행
    - 간단한 예제 구현해보기
3. **3주차**: BatchChunkSize 기능 및 PreparedBatch 심층 분석
    - `@BatchChunkSize` 애노테이션 코드 분석
    - PreparedBatch 클래스 세부 구현 파악
    - 관련 테스트 케이스 분석 및 실행
    - 현재 구현의 한계점 정리하기
4. **4주차**: 이슈 해결을 위한 설계 및 구현 계획
    - 코드 수정 상세 계획 수립
    - 관련 인터페이스 및 클래스 검토
    - 구현에 필요한 변경사항 리스트 작성
    - 초기 테스트 케이스 설계하기
5. **5주차**: 코드 구현 및 테스트
    - PreparedBatch에 청크 크기 지원 기능 구현
    - 단위 테스트 작성 및 실행
    - 기존 테스트와의 호환성 확인
    - 기능 테스트를 위한 예제 애플리케이션 구현
6. **6주차**: 코드 검토 및 PR 준비
    - 구현 코드 리팩토링 및 최적화
    - 문서화 추가 (JavaDoc, README 업데이트)
    - 기여 가이드라인에 맞게 PR 준비
    - 피드백 대응 및 최종 제출

## 7. 주요 기여 목표

- 이슈 해결을 목표로함.

이슈 #2749는 PreparedBatch 구현에서 배치 청크 크기를 직접 지원하는 기능 요청임. 현재 SQL Object 인터페이스는 @BatchChunkSize 애노테이션을 통해 한 번에 처리할 행 수를 간단히 구성할 수 있지만, PreparedBatch에서는 이러한 기능이 직접 지원되지 않음.