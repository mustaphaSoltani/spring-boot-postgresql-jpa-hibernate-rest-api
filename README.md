

# Exemple Spring Boot, PostgreSQL, JPA, Hibernate RESTful CRUD API
---

## Objectif :

Configurer Spring Boot pour utiliser la base de données PostgreSQL et créer une API RESTful 

Nous allons écrire des API REST pour une application Q & A. L'application Q & A aura deux modèles de domaine – Question et Answer. Comme une question peut avoir plusieurs réponses, nous allons définir une relation one-to-many entre l'entité Question et l'entité Answer

### Initialiser le projet :


On peut amorcer le projet Spring Boot à l'aide de l' outil Spring CLI en tapant la commande suivante dans le terminal.

``` $ spring init --name=postgres-demo --dependencies=web,jpa,postgresql postgres-demo ```


On peut également générer le projet à l'aide de l' outil Web Spring Initializr en suivant les instructions ci-dessous :

•	Aller sur http://start.spring.io .

•	Entrer postgres-demo dans le champ Artefact.

•	Ajouter Web , JPA et PostgreSQL dans la section dépendances.

•	Cliquer sur Générer un projet pour télécharger le projet.

### Configuration de PostgreSQL

Configurons Spring Boot pour utiliser PostgreSQL comme source de données  en ajoutant l'URL, le nom d'utilisateur et le mot de passe de la base de données PostgreSQL dans le fichier src/main/resources/application.properties
```bash
//Spring DATASOURCE (DataSourceAutoConfiguration & DataSourceProperties)
spring.datasource.url=jdbc:postgresql://localhost:5432/postgres_demo
spring.datasource.username= 
spring.datasource.password=
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.PostgreSQLDialect
//Hibernate ddl auto (create, create-drop, validate, update)
spring.jpa.hibernate.ddl-auto = update
```
Les deux dernières propriétés dans le fichier ci-dessus sont pour Hibernate. C'est le fournisseur JPA par défaut fourni avec spring-data-jpa.

Bien que Hibernate soit indépendant de la base de données, nous pouvons spécifier le dialecte de base de données actuel pour le laisser générer de meilleures requêtes SQL pour cette base de données.

La ddl-autopropriété est utilisée pour créer automatiquement les tables en fonction des classes d'entités dans l'application.

### Les modèles de domaine

Les modèles de domaine sont les classes qui sont mappées aux tables correspondantes dans la base de données. Nous avons deux principaux modèles de domaine dans notre application – Question et Answer. Ces deux modèles de domaine auront des champs communs d'audit communs: createdAt et updatedAt.

Il est préférable d'extraire ces champs communs dans une classe de base distincte. Nous allons créer une classe abstraite appelée AuditModel qui tiendra ces champs.

De plus, nous utiliserons la fonctionnalité de JPA Auditing de Spring Boot pour remplir automatiquement des valeurs createdAt et updatedAt lorsqu'une entité particulière est insérée / mise à jour dans la base de données.

#### 1. Model Audit 

La classe AuditModel sera étendue par d'autres entités. Il contient des annotations ``` @EntityListeners(AuditingEntityListener.class) ``` qui seront remplies automatiquement des  valeurs createdAt et updatedAt lorsque les entités sont persistantes
```bash
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@JsonIgnoreProperties( value = {"createdAt", "updatedAt"},
        allowGetters = true)
public abstract class AuditModel implements Serializable {
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "created_at", nullable = false, updatable = false)
    @CreatedDate
    private Date createdAt;

    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "updated_at", nullable = false)
    @LastModifiedDate
    private Date updatedAt;
}
```

##### Activer l'audit JPA

On ajoute à la classe principale PostgresDemoApplication.java l'annotation ``` @EnableJpaAuditing ```

#### 2. model Question 
```bash
@Entity
@Table(name = "questions")
public class Question extends AuditModel {
    @Id
    @GeneratedValue(generator = "question_generator")
    @SequenceGenerator(
            name = "question_generator",
            sequenceName = "question_sequence",
            initialValue = 1000
    )
    private Long id;
    @NotBlank
    @Size(min = 3, max = 100)
    private String title;
    @Column(columnDefinition = "text")
    private String description;
}
```

On utilise l’annotation ``` @SequenceGenerator ``` pour générer l’id de la question . On peut également utiliser la colonne PostgreSQL SERIAL en spécifiant l'annotation @GeneratedValue(strategy=GenerationType.IDENTITY). Mais un SequenceGenerator fonctionne mieux dans ce cas.

#### 3. model Answer 
```bash
@Entity
@Table(name = "answers")
public class Answer extends AuditModel {
    @Id
    @GeneratedValue(generator = "answer_generator")
    @SequenceGenerator(
            name = "answer_generator",
            sequenceName = "answer_sequence",
            initialValue = 1000
    )
    private Long id;

    @Column(columnDefinition = "text")
    private String text;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "question_id", nullable = false)
    @OnDelete(action = OnDeleteAction.CASCADE)
    @JsonIgnore
    private Question question;
}
```
La classe Answer contient une annotation ``` @ManyToOne ``` pour déclarer qu'il a une relation many-to-one avec l'entité Question.

### Les Repositories :

Les dépôts suivants seront utilisés pour accéder aux questions et réponses de la base de données.
#### 1. QuestionRepository
```bash
@Repository
public interface QuestionRepository extends JpaRepository<Question, Long> {
}
```
#### 2. AnswerRepository
```bash
@Repository
public interface AnswerRepository extends JpaRepository<Answer, Long> {
    List<Answer> findByQuestionId(Long questionId);}
```
### Construire les API REST

On écrit les API REST dans les contrôleurs pour effectuer des opérations CRUD sur les questions et les réponses.

1. QuestionController

2. AnswerController

### La classe personnalisée ResourceNotFoundException

Les API REST de question et de réponse lancent un message ResourceNotFoundExceptionlors qu'une question ou une réponse n'a pas été trouvée dans la base de données. Voici la définition de ResourceNotFoundExceptionclasse -
```bash
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
    public ResourceNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }}
 ```
La classe d'exception contient une annotation @ResponseStatus(HttpStatus.NOT_FOUND) pour indiquer à spring boot de répondre avec un statut 404 NOT FOUND lorsque cette exception est levée.

### Exécution de l'application et test des API via Postman

#### 1.	Créer une question POST /questions
 
 ![1](https://user-images.githubusercontent.com/28655112/40966352-3092a42a-68a8-11e8-98b0-ea4f9479a1b2.PNG)
 
#### 2. Obtenir des questions paginées GET /questions?page=0&size=2&sort=createdAt,desc
 
 ![2](https://user-images.githubusercontent.com/28655112/40966382-42186d74-68a8-11e8-84d3-32c613063550.PNG)
 
#### 3. Créer une réponse POST /questions/{questionId}/answers

![3](https://user-images.githubusercontent.com/28655112/40966400-51328d08-68a8-11e8-977c-bca395808f53.PNG)
 
#### 4. Obtenez toutes les réponses d'une question GET /questions/{questionId}/answers
 
 ![4](https://user-images.githubusercontent.com/28655112/40966436-60804098-68a8-11e8-85d5-e1b9aa7dc6ee.PNG)
