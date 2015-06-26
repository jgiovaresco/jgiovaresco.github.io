---
layout: post
title:  "My-PR : Sécurisation de l'application"
date:   2015-06-21 22:13:00
categories: [my-pr]
tags: [java, spring, spring security]
---

Dans cet article, je vais sécuriser l'application. 
Pour accéder à l'application, les utilisateurs devront renseigner un login/password. Pour cette étape j'utiliserai un compte utilisateur créé en dur, l'inscription des utilisateurs se fera dans l'article suivant.

Le done de cette étape est une application permettant à un utilisateur de se connecter avec un login/password.

Les sources de cette étape se trouvent sur [GitHub](https://github.com/jgiovaresco/my-pr/tree/step2) 

## Dépendances du projet

Les dépendances à ajouter sont 

 * ``compile('org.springframework.boot:spring-boot-starter-data-jpa')`` 
 	* pour l'utilisation d'une base de données
 * ``compile('org.springframework.boot:spring-boot-starter-security')`` 
 	* pour activer Spring Security
 * ``runtime("com.h2database:h2")``
 	* pour utiliser une base mémoire H2 
 * ``runtime("org.thymeleaf.extras:thymeleaf-extras-springsecurity4")``
  	* pour utiliser les tags de Spring Security dans les templates Thymeleaf
 * ``testCompile('eu.codearte.catch-exception:catch-exception:1.4.4')``
  	* librairie permettant intercepter les exceptions dans les TU 
 * ``testCompile('org.assertj:assertj-core:3.0.0')`` 
 	* librairie d'assertions utilisant une API Fluent
 * ``testCompile('org.springframework.security:spring-security-test')`` 
 	* fournit des classes utilitaires pour les tests
 * ``testCompile('org.dbunit:dbunit:2.5.1')``
  	* permet d'alimenter les tables de la base de données.
 * ``testCompile('com.github.springtestdbunit:spring-test-dbunit:1.2.1')``
  	* permet de lier Spring et DBUnit
  	
## Création des tests d'acceptation.

Cette fois les tests d'acceptation seront des tests d'intégration. L'application sera initialisée complètement et on utilisera MockMvc pour simuler les requêtes.

Pour initialiser l'application, Spring Boot me facilite la tâche. En utilisant les annotations suivantes, l'application est initialisée et le profile ``integrationTest`` est activé 

```java
@ActiveProfiles({"integrationTest"})
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = {MyPrApplication.class})
@WebAppConfiguration
```

On configure ensuite MockMvc avec le contexte Spring initialisé

```java
@Before
public void setUp()
{
	mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext)
			.addFilter(springSecurityFilterChain)
			.build();
}
```
Le principe des tests est le même pour tous

  * on interroge une url en passant éventuellement des paramètres
  * on vérifie 
    * le status de la réponse
    * l'url en cas de redirection ou une partie du contenu de la page 


### Alimentation de la base de données

Pour tester le formulaire de login, il est nécessaire d'avoir un utilisateur définit dans la base de données. Pour alimenter celle-ci, je vais utiliser [DBUnit](http://dbunit.sourceforge.net/). 
Le projet [spring-test-dbunit](http://springtestdbunit.github.io/spring-test-dbunit/) facilite l'intégration de DBUnit avec Spring.

J'aurais pu utiliser [DBSetup](http://dbsetup.ninja-squad.com/) qui propose de définir ses jeux de données à l'aide d'une API fluente. 
Cependant dans l'étape suivante je vais devoir contrôler le contenu de la base de données après exécution des tests. Or DBSetup ne permet que l'alimentation d'une base de données. 

#### Déclaration DBUnit

Pour utiliser DBUnit dans un test il faut ajouter l'annotation suivante 

```java
@TestExecutionListeners({DependencyInjectionTestExecutionListener.class,
                         TransactionalTestExecutionListener.class,
                         DbUnitTestExecutionListener.class})
```

Il est nécessaire d'ajouter les listeners ``DependencyInjectionTestExecutionListener.class`` et ``TransactionalTestExecutionListener.class`` car dès qu'un ``TestExecutionListeners`` est déclaré, les listeners définis par défauts sont oubliés.
Sans cette déclaration, il ne serait plus possible d'injecter des beans dans la classe de test.

#### Déclaration d'un jeu de données

La déclaration d'un jeu de données se fait grâce à l'annotation ``@DatabaseSetup``

```java
@DatabaseSetup("users.xml")
```

Le fichier passé dans l'annotation est récupéré de la même manière qu'un fichier de configuration Spring déclaré avec l'annotation ``@ContextConfiguration``.

### Fonctionnement du formulaire de connexion

Les tests se trouvent dans la classe ``fr.mypr.security.controller.FormLoginIT``.

Ci dessous un extrait 

```java
@Test
public void login_should_redirect_to_login_form_when_invalid_password() throws Exception
{
	// @formatter:off
	mockMvc.perform(
			post("/login/authenticate")
					.param(REQUEST_PARAM_EMAIL, IntegrationTestConstants.User.REGISTERED_USER.getEmail())
					.param(REQUEST_PARAM_PASSWORD, INVALID_PASSWORD)
			)
//				.andDo(print())
			.andExpect(status().isFound())
			.andExpect(redirectedUrl("/login?error=bad_credentials"));
	// @formatter:on
}
```

### Contrôle de l'authentification

Pour contrôler l'authentification, on va vérifier que la page Index affiche
  
  * un lien pour se connecter lorsque l'utilisateur n'est pas connecté
  * un lien pour se déconnecter lorsque l'utilisateur est déjà connecté

On peut simuler un utilisateur connecté grâce aux méthodes utilitaires de la classe ``org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors``.

L'exemple suivant montre comment effectuer une requête avec un utilisateur connecté.

```java
@Test
public void showIndexPage_asRegisteredUser_should_render_index_page_with_logout_link() throws Exception
{
	// @formatter:off
	mockMvc.perform(get("/")
		   .with(user(IntegrationTestConstants.User.REGISTERED_USER.getEmail()))
			)
//			.andDo(print())
			.andExpect(status().isOk())
			.andExpect(view().name("index"))
			.andExpect(content().string(containsString("<a href=\"/logout\">Log Out</a>")));
	// @formatter:on
}
```

## Le modèle de données

Les comptes utilisateurs sont définis par la classe ``fr.mypr.user.model.UserAccount``. Il s'agit d'un bean JPA contenant les propriétés classiques définissant un compte utilisateur.

L'accès à ces comptes utilisateurs se fait avec la classe ``fr.mypr.user.repository.UserAccountRepository`` dans laquelle est définie une méthode permettant de trouver un compte grâce à l'attribut ``email``.

## Spring Security

### Fonctionnement de Spring Security

Lors d'une authentification via formulaire, Spring Security utilise le _username_ renseigné dans le formulaire pour retrouver l'utilisateur.
La recherche de l'utilisateur se fait en appelant la méthode ``loadUserByUsername`` d'un service implémentant l'interface ``org.springframework.security.core.userdetails.UserDetailsService``.

Si la méthode trouve un utilisateur, Spring Security s'occupe de vérifier le mot de passe et si tout est bon, il considère l'utilisateur authentifié.
Dans le cas où l'utilisateur n'est pas trouvé ou si le mot de passe est incorrect, l'authentification est interrompue.

### Configuration de Spring Security

La configuration de Spring Security peut se faire avec une classe annotée avec ``@Configuration`` et ``@EnableWebSecurity``. Pour simplifier la configuration
on étend la classe ``org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter``.

La déclaration du service à utiliser pour trouver les utilisateurs se fait en redéfinissant la méthode ``void configure(AuthenticationManagerBuilder auth)``

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception
{
	auth
			.userDetailsService(userDetailsService())
			.passwordEncoder(passwordEncoder());
}

@Bean
public PasswordEncoder passwordEncoder()
{
	return NoOpPasswordEncoder.getInstance();
}

@Bean
public UserDetailsService userDetailsService()
{
	return new RepositoryUserDetailsService(userAccountRepository);
}
```

On peut remarquer qu'il n'y a aucune protection pour les mots de passes (pas de chiffrement). 

En redéfinissant la méthode ``void configure(WebSecurity web)``, il est possible de déclarer des chemins pour lesquels la sécurité doit être désactivée (CSS, JS, ...).
 
```java
@Override
public void configure(WebSecurity web) throws Exception
{
	// @formatter:off
	web
		.ignoring()
			.antMatchers("/css/**", "/js/**", "/favicon.ico", "/**/*.png", "/**/*.gif", "/**/*.jpg");
	// @formatter:on
}
```

La configuration de la sécurité se trouve dans la méthode ``void configure(HttpSecurity http)``

```java
@Override
protected void configure(HttpSecurity http) throws Exception
{
	// @formatter:off
	http
		.csrf()
			.disable()
		.formLogin()
			.loginPage("/login")
			.loginProcessingUrl("/login/authenticate")
			.failureUrl("/login?error=bad_credentials")
			.usernameParameter("email")
		.and()
			.logout()
				.deleteCookies("JSESSIONID")
				.logoutUrl("/logout")
				.logoutSuccessUrl("/")
		.and()
			.authorizeRequests()
				.antMatchers(
						"/", "/login"
				).permitAll()
				.antMatchers("/**").hasRole("USER")
		;
	// @formatter:on
}
```

  * La sécurité CSRF est désactivée
  * L'authentification se fait avec un formulaire
  	* la page contenant le formulaire d'authentification est ``/login``
  	* l'url permettant l'authentification est ``/login/authenticate``
  	* l'url en cas d'échec de l'authentification est ``/login/?error=bad_credentials``
  	* le paramètre correspondant à l'username dans le formulaire est ``email``.
  * Pour la déconnexion
  	* le cookie ``JSESSIONID`` est supprimé
  	* l'url de déconnexion est ``/logout``
  	* l'url à afficher lorsque la déconnexion réussie est ``/``
  * La sécurisation des requêtes est
    * les urls ``/`` et ``/login`` sont accessibles par tout le monde
    * tandis que toutes les autres urls sont accessibles pour les utilisateurs ayant le role ``USER``

## Problème(s) rencontré(s)

Le 1er problème rencontré fut avec [Lombok](https://projectlombok.org/). Les annotations n'étaient pas traitées par Intellij. 
Pour activer leur traitement il faut bien vérifier que l'option **Enable annotation processing** soit coché dans la configuration : _Build, Execution, Deployment > Compiler > Annotation Processor_

Le 2e problème est liés à l'utilisation de tags Spring Security dans les templates Thymeleaf. Pour les activer il est nécessaire d'ajouter la dépendance ``runtime("org.thymeleaf.extras:thymeleaf-extras-springsecurity4")``