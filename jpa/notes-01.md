# JPA 
## ORM 
### Why ORM exists? 
A player table called `player` has one row.
``` 
player
------
id | username | balance
1  | rajat    | 500.00
``` 

A corresponding player object looks as below.
```java
class Player { Long id; String username; BigDecimal balance; }
``` 

- We have a row in the database disk and a corresponding object on the Java side.
- Each of these can't read each other, as they are sitting in different servers/systems. 
- Something must pass values between these two and remember which object belongs to which role. 

-- Revisit persistence context. 

### Impedance mismatch

**Granularity**
- 
- A Java object can have a nested object inside it. We club related objects in a sub-object and place it inside another object, like a `client` having an  `address` or `personaldetails`
- This is very common in the Java world, instead of creating one giant object, we club related objects into their own classes. 
- However in the database world, it expects the related objects to be present in the same row instead of creating a separate table. It's overkill on the database side of things. 
- The ORM, which is bridging the gap between Java and the relational world needs to do some extra work. 
- It can flatten all those objects into one row using `@Embeddable`/`@Embedded`

**Inheritance**
- 
- Object-oriented side of things has inheritance. The parent/child class can have its own fields. 
- But for the relational side of things there is no inheritance concept. 
- There will be only one table for all the subtypes of the `Quiz`. And it can't have a parent type in the table which is abstract. 
- Hence there are three possible options now. 
  - A single table with a subtype
  - A joined table having a common `quiz` table + two different `skillQuiz` and `timeQuiz` tables 
  - And one table per subtype 
- To conclude there is no clean answer but a trade-off that we'll have to accept. 

**Identity**
- 
- In the Java world we have two variations of `equals`. 
  - `==` which checks for reference equality. 
  - `.equals()` which checks for value equality.
- On the relational side it is entirely different. It all revolves around the primary key. Two rows are the same if their primary key is the same. 
- So when we fetch (say using JDBC) two objects with PK value as 5. Even though they are same rows in the database. They live as two different objects on the Java side.  
```java 
Quiz a = jdbcEngine.find(Quiz.class, 42L);
Quiz b = jdbcEngine.find(Quiz.class, 42L); 
```
- And if we modify any of the rows with different values, we have two copies for the same row on the Java side, which causes bugs. 
- This is why persistence context exists - which ensures within the same entityManager session requesting the same route twice will create only one Java object not two.
- This way Hibernate manages identity mismatch. But this is valid only if we have one session. If we have got multiple sessions then there is no guarantee of a single object being created. 

**Associations**
-
- In the OOPs world the reference to another object is directional and typed. `quiz.getQuestion()` Returns a `List<Question>`, And there is no inherent reverse. Unless we explicitly code one `question.getQuiz()`.
- Even then there are two different references that are attributing it to which are independently pointing to related but different objects. 
- In the relational world, the association is non-directional and untyped. The `question` table has a foreign key to the `quiz` table. And the `quiz` table has no idea about the `question` table. However the same foreign key column can be used to query both sides.

Direction A — "give me all questions for quiz 42":
```sql
SELECT * FROM question WHERE quiz_id = 42;
```
Direction B — "given question 1, what quiz is it in?":
```sql
SELECT q.*
FROM quiz q
       JOIN question qn ON qn.quiz_id = q.id
WHERE qn.id = 1;
```
- Hence as a developer we need to decide on the Java side which code owns the relationship. Which is generally mapped using the `mapped_by` param of the annotation

//revisit this ↓ 
```java
@Entity
class Quiz {
    @Id Long id;
    
    @OneToMany(mappedBy = "quiz")  // NOT the owning side — just a view
    List<Question> questions = new ArrayList<>();
}

@Entity
class Question {
    @Id Long id;
    
    @ManyToOne              // the OWNING side — this is what controls quiz_id
    @JoinColumn(name = "quiz_id")
    Quiz quiz;
    
    String text;
}
```
`mappedBy = "quiz"` on the Quiz side is JPA saying: "the questions collection is just a read-view; it does not control the quiz_id column. Only Question.quiz controls that column."

If you only update quiz.questions.add(newQuestion) and forget newQuestion.setQuiz(quiz), the foreign key never gets set, because JPA persists based on the owning side's field, not the collection

The final table state will be:
```
quiz table:
id | title
42 | "Capital Cities Trivia"


question table:
id | quiz_id | text
1  | NULL    | "What is the capital of Japan?"
```

**Navigation** 
-
- In the Java world reaching a nested or referencing object is straightforward all we'll have to do is keep on doing that operation. 
```java
quiz.getQuestions().get(0).getCorrectAnswer().getText();
```
- However in the relational world, we have to join each table to navigate into its contents. Which is a costly operation and can mean a new network call going on to DB.