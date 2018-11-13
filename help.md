The core functionality of Spring Data REST is to export resources for Spring Data repositories.
Spring Data REST exposes sub-resources of every item resource for each of the associations the item resource has. By default, Spring Data REST uses HAL (hypertext application language) to render responses.

RepositoryRestMvcConfiguration - the @Configuration for Spring Data REST
RepositoryRestConfigurerAdapter - used to customize the configuration

By default, Spring Data REST serves up REST resources at the root URI, "/". Register a @Bean which extends RepositoryRestConfigurerAdapter and overwrite configureRepositoryRestConfiguration in order to change it. With spring boot use spring.data.rest.basePath=/new-base-path-for-spring-data-rest in application.properties.
http://ip:port/base-path-for-spring-data-rest returns a JSON with the first level of resources provided with Spring Data REST.

@RepositoryRestResource - used on a Repository interface to customize the path and name of the resource
@RestResource - used on a method to customize the path and name of the exported resource; use exported = false in order not to export the resource

Annotation org.springframework.data.rest.core.config.Projection used to define altered views for entities (e.g.: not showing the password for user).
URI would look like: http://localhost:8080/persons/1?projection=noPassword
@Projection can be used with SpEL expression to create new properties.
An excerpt is a projection that is applied to a resource collection automatically, e.g.:
@RepositoryRestResource(excerptProjection = NoPassword.class)
interface PersonRepository extends CrudRepository {}

Validation is achived using a registered org.springframework.validation.Validator instance:
a. beforeCreatePersonValidator -> prefix the bean name with the Data REST event name
b. assigning Validators manually -> override RepositoryRestMvcConfiguration.configureValidatingRepositoryEventListener and add the validators

Events: BeforeCreateEvent, AfterCreateEvent, BeforeSaveEvent, AfterSaveEvent, BeforeLinkSaveEvent, AfterLinkSaveEvent, BeforeDeleteEvent, AfterDeleteEvent. Writing an ApplicationListener:
a. extend AbstractRepositoryEventListener (onBeforeSave, etc) -> makes no distinction based on the type of the entity
b. write an annotated handler -> @RepositoryEventHandler on class and @HandleBeforeSave on method (first parameter determines the domain type whose events you’re interested)

Creating links from Spring MVC controllers to Spring Data REST resources is possible with injectable RepositoryEntityLinks (equivalent to EntityLinks from Spring HATEOAS): entityLinks.linkToCollectionResource(Person.class).

ALPS (Application-Level Profile Semantics) is a data format for defining simple descriptions of application-level semantics.
ALPS (application/alps+json or application/schema+json) -> used for metadata provided by a Spring Data REST-based application.
http://ip:port/base-path-for-spring-data-rest/profile -> metadata location
ALPS list first entity's attributes (+ projections) then operations.

Spring security works fine with Spring Data REST; just use @PreAuthorize as usual.

The developer of the HAL spec has a useful application: the HAL Browser.

Write a custom handler (overriding Spring Data REST response handlers) for a specific resource using @RepositoryRestController (instead of @RestController). There's a custom HandlerMapping instance that responds only to the RepositoryRestController which is considered before the one for @Controller or @RestController.

Use ResourceProcessor interface in order to provide links to other resources from a particular entity. You declare spring beans implementing ResourceProcessor for a specific entity.
If you register your own ConversionService in the ApplicationContext and register your own Converter, then you can return a Resource implementation of your choosing.

Maven dependency
<dependency>
  org.springframework.data
  spring-data-rest-webmvc
  2.4.2.RELEASE
</dependency>
