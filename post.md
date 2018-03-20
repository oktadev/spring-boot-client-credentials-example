---
title: Secure Server to Server Communication with Spring Boot and OAuth 2.0
tags: spring, oauth, oauth2, java
author: bdemers
Tweets:
- Secure Server to Server Communication with Spring Boot and OAuth 2.0
- OAuth Client Credentials Grant with Spring Boot
---

Most of the OAuth 2.0 guides I've seen are usually focus around the context of a user, i.e. login to a application using Google, Github, Okta, etc, then do something on behalf of that user. This leaves out server-to-server communication where there is no user, you just have one service connecting to another one.
 
 -This is very similar to a database connection in your application where you probably have a service account to access said database, but instead of a database you have an API endpoint.-

## OAuth 2.0 Client Credentials Grant

The goal of the [client credentials](https://tools.ietf.org/html/rfc6749#section-4.4) grant is to allow two machines to communicate securely. You have a client (think of this as your application) making API requests to another service (this is your resource server).  Before we dig into why this is important, let's take a step back and talk about what we did before OAuth. If you are an OAuth pro you can skip ahead to the [code examples below](#TODO) or check out the example on [GitHub](#TODO).

Before OAuth, one common scenario was to create a service account, the client would then send the accounts username and password when making API requests.  The API server would connect to a user store (database, LDAP, etc) in order to validate the username and password.

TODO: insert sequence diagram here

This has a few draw backs and exposure points:

- Each application in the diagram above handles the username and password
- A second username and password might be needed to connect to user store
- The same username and password is used for each request

There are ways to help mitigate some of the risk, but that is for a different blog post.

The OAuth client credentials grant spins this around, the client uses `client_id` and `client_secret` in place of a username/password but instead sending them directly to your API server you exchange them with an [authorization server](https://tools.ietf.org/html/rfc6749#section-1.1).

TODO: insert sequence diagram here

The authorization server returns a temporary access token (which is reused until it expires), the client then uses this access token when communicating with the resource server. The resource server then validates the token with the authorization server.

I'll talk about a couple ways to reduce number of calls further at the end of this post, but first, on to the example!

## Example Time!

Enough about the reasons why, I want to talk about the how you actually implement it with Spring using two applications a `client` and `server`.  The server will will have a single endpoint which returns a "message of the day".  The client is just a simple command line application, you could easily replace this with your own web application.

### Setup your Authorization Server

This is an Okta blog post, so I'll of course show you how to use Okta, but you could use any OAuth authorization server.

If you don't already have a free developer account head over to [developer.okta.com](https://developer.okta.com/), just click the sign up button and fill out the form. When that’s done you’ll have two things, your Okta base URL, which looks something like: `dev-123456.oktapreview.com` and an email with instructions on how to activate your account.

After activating your account and while you are still in the Okta Developer Console you need to create an application and a custom OAuth scope. The application will give you a client id and secret, while the custom scope will restrict your access token to this example. 

Click the **Applications** menu item then the **Add Application** button, then the **Service** -> **Next**
Change the name to whatever you want (I'm going to use "My MOD App"), then press **Done**.

TODO: create image

You will need the **Client ID** and **Client secret** values for the next steps.

The final thing you need to do is create a [custom scope](https://www.oauth.com/oauth2-servers/scope/defining-scopes/) for your application. 

From the menu bar select **API** -> **Authorization Servers**.  If you are new to Okta you have one already click on the edit pencil button, then click **Scopes** -> **Add Scope**.  Fill out the name field with `custom_mod` and press the **Create** button.

Remember the **Issuer URI** value, you will need this in the next steps.

On to the fun stuff!

### Create a Resource Server

As I mentioned before my resource server is going to be overly simple and just contain a single `/mod` endpoint. Create a new \project using the [Spring Initializer](https://start.spring.io/) on the command line:  

```bash
curl https://start.spring.io/starter.tgz  \
  -d artifactId=creds-example-server \
  -d dependencies=security,web \
  -d language=java \
  -d type=maven-project \
  -d baseDir=creds-example-server \
| tar -xzvf -

# change into the new directory
cd creds-example-server
```

You will also need to manually add one more dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.security.oauth.boot</groupId>
    <artifactId>spring-security-oauth2-autoconfigure</artifactId>
    <version>2.0.0.RELEASE</version>
</dependency>
```

I also renamed the `DemoApplication` to `ServerApplication` because we are going to create another application shortly.

Update `ServerApplication` to include the `@EnableResourceServer` annotation and add a simple REST controller:

```java
@EnableResourceServer
@SpringBootApplication
public class ServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServerApplication.class, args);
    }

    /**
     * Allows for @PreAuthorize annotation processing.
     */
    @EnableGlobalMethodSecurity(prePostEnabled = true)
    protected static class GlobalSecurityConfiguration extends GlobalMethodSecurityConfiguration {
        @Override
        protected MethodSecurityExpressionHandler createExpressionHandler() {
            return new OAuth2MethodSecurityExpressionHandler();
        }
    }

    @RestController
    public class MessageOfTheDayController {
        @GetMapping("/mod")
        @PreAuthorize("#oauth2.hasScope('custom_mod')")
        public String getMessageOfTheDay(Principal principal) {
            return "The message of the day is boring for user: " + principal.getName();
        }
    }
}
```

Last thing to do is configure our application! I renamed my `application.properties` file to `application.yml` and updated it to include:

```yml
security:
  oauth2:
    client:
      clientId: {client-id-from-above}
      clientSecret: {client-secret-from-above}
    resource:
      tokenInfoUri: {issuer-uri-from-above}/v1/introspect
```

That is it, few lines of code and a couple lines of config! Spring Boot will automatically handle the validation of the access tokens, all you need to worry about is your code!  

Start it up and leave it running:

```bash
./mvn spring-boot:run
```

You can try to access `http://localhost:8080/mod` if you want, it will respond with a `401`.

### Create the Client

As mentioned above, I'm going to create a simple command line client to keep things simple but you could just as easily duplicate this logic in any type of application.  

Open up another terminal window and create a second application with the Spring Initializer:

```bash
curl https://start.spring.io/starter.tgz  \
  -d artifactId=creds-example-client \
  -d dependencies=security \
  -d language=java \
  -d type=maven-project \
  -d baseDir=creds-example-client \
| tar -xzvf -

# change into the new directory
cd creds-example-client
```

Same as before add in the Spring OAuth library as a dependency in your `pom.xml`: 

```xml
<dependency>
    <groupId>org.springframework.security.oauth.boot</groupId>
    <artifactId>spring-security-oauth2-autoconfigure</artifactId>
    <version>2.0.0.RELEASE</version>
</dependency>
```

This time I'm to start with the configuration (again I renamed `application.properties` to `application.yml`):

```yml
example:
    baseUrl: http://localhost:8080
    oauth2:
        client:
            grantType: client_credentials
            clientId: {client-id-from-above}
            clientSecret: {client-secret-from-above}
            accessTokenUri: {issuer-uri-from-above}/v1/token
            scope: custom_mod
```

I've namespaced the configuration under `example` as your could be connecting to multiple servers.

I configured a few properties:

- `baseUrl` is the base URL of our example server
- `grantType` defines the grant type for the connection
- `clientId` and `clientSecret` are the same as above
- `accessTokenUri` defines the URI used to get an access token
- `scope` is our custom scope we created above

Last up is our `ClientApplication` (renamed from `DemoApplication`):

```java
@SpringBootApplication
public class ClientApplication implements CommandLineRunner {

    private final Logger logger = LoggerFactory.getLogger(ClientApplication.class);

    @Value("#{ @environment['example.baseUrl'] }")
    private String serverBaseUrl;

    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }

    @Bean
    @ConfigurationProperties("example.oauth2.client")
    protected ClientCredentialsResourceDetails oAuthDetails() {
        return new ClientCredentialsResourceDetails();
    }

    @Bean
    protected RestTemplate restTemplate() {
        return new OAuth2RestTemplate(oAuthDetails());
    }

    @Override
    public void run(String... args) {
        logger.info("MOD: {}", restTemplate().getForObject(serverBaseUrl + "/mod", String.class));
    }
}
```

There are a few things I want to touch on:

- The `CommandLineRunner` interface adds a `run` method, it is called automatically after initialization, the application exits after leaving this method
- I created a `ClientCredentialsResourceDetails` bean which is bound to my configuration properties: `example.oauth2.client`
- I use an `OAuth2RestTemplate` in place of a standard `RestTemplate` this automatically manages all of the OAuth access token exchange and sets the `Authentication: Bearer` header value.  Basically, it handles all of the OAuth detail so I don't need to worry about any of them!

## Extra Credit: Reduce the Number of Calls to the Authorization Server

The second sequence diagram above seems more complicated then the first, even when factoring in the reuse of an access token. Access tokens are opaque, there is no spec behind them, the format is left to the implementation of the authorization server.  At Okta we use signed JWTs which means you can [validate them locally](/authentication-guide/tokens/validating-access-tokens) instead of making an additional request.

TODO: insert sequence diagram

We have helper libraries in a [few different languages](https://github.com/okta?utf8=%E2%9C%93&q=okta-jwt-) and a [Spring Boot starter](https://github.com/okta/okta-spring-boot) that will handle the local validation for you.

**NOTE:** at the time of this writing `okta-spring-boot` only works with Spring Boot 1.5.x. 

## Conclusion

In this post I've explained the OAuth client credentials grant type and created tiny demo applications that exercised this flow (with very little code, thanks to Spring Boot!). If you have questions leave them below or ping me ([@briandemers](https://twitter.com/briandemers)) or [@OktaDev](https://twitter.com/oktadev) on Twitter.  

For more info on OAuth and Okta checkout these posts:

- [What the Heck is OAuth?](/blog/2017/06/21/what-the-heck-is-oauth)
- [OAuth.com](https://www.oauth.com/)
- [Secure your SPA with Spring Boot and OAuth](/blog/2017/10/27/secure-spa-spring-boot-oauth)

