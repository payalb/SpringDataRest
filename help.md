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


Overriding Spring Data REST Repositories
Will Faithfull | 03 May 2017
We often use spring-data-rest to abstract away simple RESTful CRUD boilerplate, and it is very good at it. The problem comes in more complicated situations. Sometimes you need to step in and override one of the request handlers to do something slightly differently, or trigger an event. So, how do you reliably override request handlers for repositories which are exported with @RepositoryRestResource?

The way to do so is poorly documented, and contains several caveats, which I will share from my experience. All we get in the documentation is this little snippet in section 16.4.

@RepositoryRestController
public class ScannerController {

    private final ScannerRepository repository;

    @Autowired
    public ScannerController(ScannerRepository repo) { 
        repository = repo;
    }

    @RequestMapping(method = GET, value = "/scanners/search/listProducers") 
    public @ResponseBody ResponseEntity<?> getProducers() {
        List<String> producers = repository.listProducers(); 

        //
        // do some intermediate processing, logging, etc. with the producers
        //

        Resources<String> resources = new Resources<String>(producers); 

        resources.add(linkTo(methodOn(ScannerController.class).getProducers()).withSelfRel()); 

        // add other links as needed

        return ResponseEntity.ok(resources); 
    }

}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
What is wrong with this? Just one thing really. It adds a new method to the repository route, rather than overriding an existing one. Importantly, it doesn't mention that this approach will expressly not allow you to override existing handlers. So, how do we define a controller which takes precedence over a repository in the event that both sit on the same path?

From the top..
Let's say we have an exported FooRepository.

@RepositoryRestResource(exported=true, path="foos")
public interface FooRepository extends PagingAndSortingRepository<Foo,Long> {

}
1
2
3
4
This exposes the basic CRUD:

GET /foos
POST /foos
GET /foos/{id}
PUT /foos/{id}
POST /foos/{id}
PATCH /foos/{id}
DELETE /foos/{id}
HEAD /foos/{id}
Along with any association links rendered.

Let's say we don't want to return ALL Foos on GET /foos, but just the ones associated with our member.

How to do it
We make an overriding controller.

@BasePathAwareController
public class FooController {

    private final FooRepository fooRepository;

    public FooController(final FooRepository fooRepository) {
        this.fooRepository = fooRepository;
    }

    @RequestMapping(path="foos", method=RequestMethod.GET, produces="application/hal+json")
    public Resources getAllFooFiltered() {
        // Do your filtering and end up with a HATEOAS resources to return
    }

}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
That's it - a request to /foos will hit your controller now. Any other requests in the table above will still go to the exported repository handlers.

Gotchas / FAQ
It must be @BasePathAwareController. @RepositoryRestController will not override exported handlers, but you can use it to add handlers that aren't exported on the repository.
You must NOT have a @RequestMapping at the type level - it has to be at the method level as above.
The path in your @RequestMapping(path="...") must NOT start with a /
OK: @RequestMapping(path="foos")
NOT OK: @RequestMapping(path="/foos")
You cannot define a standalone mapping like foos/count, because it clashes with foos/{id}. If you need to do this, you must override  foos/{id} and decide what do do based on the {id} path variable content. (e.g. if "count".equals(id) { ... }). This is sort of an antipattern anyway.
You must NOT have any @PreAuthorize annotations on the type or methods - this causes the class to be proxied and prevents the desired behaviour. If you need security rules, implement them in a service layer below the controller, or wait for Spring to fix. DATAREST-535
Appendix - @PreAuthorize and controllers
Attempting to apply @PreAuthorize annotations on controller methods has been the cause of a lot of mysterious behaviour. A primary issue is the difference between CGLIB and JDK Dynamic proxies. JDK dynamic proxies are what spring uses by default if you have  @EnableAspectJAutoProxy - but they only work for classes that are interfaced. CGLIB proxies are more clever. You enable them with  @EnableAspectJAutoProxy(proxyTargetClass=true), and they will actually create a dynamic subclass of the targeted type at runtime, thus allowing proxies of interface-less classes. However, this doesn't play nice with Spring Data REST, for reasons which I have not yet had time to investigate.

If we apply some security annotations to the above overriding controller, and then try to hit one of the repository methods, what happens?

{
  "timestamp": "2017-05-04T08:23:46.090+0000",
  "status": 405,
  "error": "Method Not Allowed",
  "exception": "org.springframework.web.HttpRequestMethodNotSupportedException",
  "message": "Request method 'PATCH' not supported",
  "path": "/foos/1"
}
1
2
3
4
5
6
7
8
So, that's now no good. We can offer our own routes, but can't hit the existing repository routes.

Part of the reason this is particularly prevalent in controller classes is that they almost never implement interfaces, so their dynamic proxies (for  @PreAuthorize or @Secured) will be CGLIB. The bulletproof solution at the moment is to do your fancy authorization annotations at the service layer. In fact, this is what is advocated in the spring security documentation itself (important part italicised at the bottom).

44.2.16 I have added Spring Security’s element to my application context but if I add security annotations to my Spring MVC controller beans (Struts actions etc.) then they don’t seem to have an effect.

In a Spring web application, the application context which holds the Spring MVC beans for the dispatcher servlet is often separate from the main application context. It is often defined in a file called myapp-servlet.xml, where "myapp" is the name assigned to the Spring DispatcherServlet in web.xml. An application can have multiple DispatcherServlets, each with its own isolated application context. The beans in these "child" contexts are not visible to the rest of the application. The"parent" application context is loaded by the ContextLoaderListener you define in your web.xml and is visible to all the child contexts. This parent context is usually where you define your security configuration, including the element). As a result any security constraints applied to methods in these web beans will not be enforced, since the beans cannot be seen from the DispatcherServlet context. You need to either move the declaration to the web context or moved the beans you want secured into the main application context.

Generally we would recommend applying method security at the service layer rather than on individual web controllers.
