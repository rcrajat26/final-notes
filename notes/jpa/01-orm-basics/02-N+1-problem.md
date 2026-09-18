# N + 1 problem 

### Background
- Every time the application queries a database, it's not free. It's a network call that goes out. So it costs some time. 
- The more the number of queries, the more is the cost in terms of time. 

### Table under use 
```
quiz table:
id | title
1  | "Capital Cities"
2  | "Movie Trivia"
3  | "Science Basics"

question table:
id | quiz_id | text
1  | 1        | "Capital of Japan?"
2  | 1        | "Capital of France?"
3  | 2        | "Who directed Jaws?"
4  | 3        | "Speed of light?"
5  | 3        | "What is H2O?"
```
Each quiz has multiple questions in a one-to-many relationship. 

### Problem
- Let's say we now query - "show me all quizzes, with their question count"
- We'll have to loop over all the quizzes and all the questions those quizzes have.

```java
List<Quiz> quizzes = entityManager
    .createQuery("SELECT q FROM Quiz q", Quiz.class)
    .getResultList();

for (Quiz quiz : quizzes) {
    System.out.println(quiz.getTitle() + ": " + quiz.getQuestions().size());
}
```

### What happens under the hood? 
- The first query will fetch all the quizzes.
```sql
SELECT * FROM quiz;
```

```
id | title
1  | "Capital Cities"
2  | "Movie Trivia"
3  | "Science Basics"
```

- That's one query and we have three quizzes in the memory. At this point we don't have any awareness about the questions that each quiz has, as we queried only the quiz table.
- Now we start looping. We ask questions of Quiz-1, by default JPA mark `@OneToMany` associations as lazy. Meaning don't fetch anything till someone asks for it specifically. 
- When we fetched from the quiz query, the questions array came out to be empty-looking proxy 
- When we call `quiz.getQuestions().size()`, JPA will now fire a query to fetch the questions for that quiz. 

Query #2 — for quiz 1:
```sql
SELECT * FROM question WHERE quiz_id = 1;
```

```
id | quiz_id | text
1  | 1       | "Capital of Japan?"
2  | 1       | "Capital of France?"
```

- Similarly for the second quiz, JPA makes a call to get Quiz-2's questions.
Query #3 — for quiz 2:
```sql
SELECT * FROM question WHERE quiz_id = 2;
```

```
id | quiz_id | text
3  | 2       | "Who directed Jaws?"
```

- And for the third quiz, JPA makes a call to get Quiz-3's questions.
Query #4 — for quiz 3:
```sql
SELECT * FROM question WHERE quiz_id = 3;
```

```
id | quiz_id | text
4  | 3       | "Speed of light?"
5  | 3       | "What is H2O?"
```

### Observation 
- 1 query to get all the quizzes.
- N queries (one per quiz) to get each quiz's questions — here N = 3, so 3 more queries.
- Total: 1 + 3 = 4 queries, to answer what conceptually is one simple question ("how many questions does each quiz have").
- Hence we call this the N+1 problem, N being the number of rows returned by the first query
- Once N grows really big the queries start suffering from the lag.

### Fix A — a JOIN FETCH, one query total:
```java
List<Quiz> quizzes = entityManager
    .createQuery("SELECT q FROM Quiz q JOIN FETCH q.questions", Quiz.class)
    .getResultList();
    
//Springboot way
public interface QuizRepository extends JpaRepository<Quiz, Long> {

    @Query("SELECT q FROM Quiz q JOIN FETCH q.questions")
    List<Quiz> findAllWithQuestions();

    @Query("SELECT q FROM Quiz q JOIN FETCH q.questions WHERE q.id = :id")
    Optional<Quiz> findByIdWithQuestions(@Param("id") Long id);
}
```

this generates below sql statement:
```sql
SELECT q.*, qn.* 
FROM quiz q 
JOIN question qn ON qn.quiz_id = q.id;
```

```
id | title            | id | quiz_id | text
1  | "Capital Cities"  | 1  | 1       | "Capital of Japan?"
1  | "Capital Cities"  | 2  | 1       | "Capital of France?"
2  | "Movie Trivia"    | 3  | 2       | "Who directed Jaws?"
3  | "Science Basics"  | 4  | 3       | "Speed of light?"
3  | "Science Basics"  | 5  | 3       | "What is H2O?"
```

Hibernate takes this single flat result set and reconstructs it back into 3 Quiz objects, each with its questions collection already populated — no further trips needed, no matter how many quizzes there are. 1 query total, regardless of N.

### Fix B — batch fetching
- a middle ground: instead of one query per quiz, Hibernate can be configured (@BatchSize) to fetch questions for, say, 20 quizzes at a time:

```sql
SELECT * FROM question WHERE quiz_id IN (1, 2, 3, ..., 20);
```

This turns N individual queries into roughly N/20 queries — much better than N+1, though not as good as the single JOIN FETCH.

### Note
- If JOIN FETCH fixes it so cleanly, why isn't everything eager by default? Because eager fetching has its own cost: if you only needed the quiz titles for a dropdown list and never touched .getQuestions() at all, that JOIN just made your query heavier and returned duplicate quiz data (notice "Capital Cities" repeated across rows above) for no reason. Lazy loading exists specifically so you don't pay the join cost when you don't need the related data

### Conclusion/Definition

N+1 happens when you fetch a list of parent rows in one query, then — because the association is lazy — trigger a separate query for each parent's children as you loop over them, turning what should be 1 or 2 queries into 1 + (number of parents), with no compiler warning or obvious code smell to flag it.

---
### Miscellaneous

The n+1 problem can be solved by using FetchType.EAGER
```java
@Entity
class Quiz {
    @Id Long id;
    
    @OneToMany(mappedBy = "quiz", fetch = FetchType.EAGER)  // changed from LAZY
    List<Question> questions;
}
```

- All this does is change the time of loading not the way of loading. And instead of waiting for `.getQuestions()` to be called, it will fetch during the load time itself of the first query. 
- Hibernate still continues to query N+1 queries. Hence it doesnt solve anything the fix is still to use `JOIN FETCH`
- A few concrete reasons EAGER tends to be actively discouraged rather than just neutral: It's global, not contextual. FetchType.EAGER is set once on the entity mapping and applies to every query that loads a Quiz, everywhere in the app, forever. But whether you need questions loaded is a per-use-case decision — a dropdown listing quiz titles doesn't need it; a "review quiz" screen does. EAGER can't tell these apart; it always loads, everywhere, even in the 90% of cases that didn't want it.
