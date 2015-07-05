---
layout: post
title:  "My-PR : Inscription des utilisateurs"
date:   2015-06-30 15:38:00
categories: [my-pr]
tags: [java, spring]
---

Dans cet article, j'ajoute la fonctionnalité d'inscription des utilisateurs.

Pour s'inscrire les utilisateurs pourront créer un compte en utilisant un formulaire d'inscription.

Le done de cette étape est une application permettant à un utilisateur de s'inscrire pour pouvoir ensuite accéder à l'application avec un login/password.
Les données sont toujours stockées dans une base H2.

Les sources de cette étape se trouvent sur [GitHub](https://github.com/jgiovaresco/my-pr/tree/step3) 

## Création des tests d'acceptation.

Comme précédemment, j'utilise DBUnit pour alimenter la base de données. 
De plus je vais utiliser les annotations de [spring-test-dbunit](http://springtestdbunit.github.io/spring-test-dbunit/) pour vérifier l'état de la base à la fin des tests. 

Pour contrôler l'état de la base à la fin d'un test, il suffit d'annoter le test avec ``@ExpectedDatabase`` en lui passant en paramètre un fichier XML des données attendues. 

### Affichage du formulaire d'inscription

La première chose à faire est d'afficher le formulaire d'inscription. 

```java
@Test
@DatabaseSetup("no-users.xml")
@ExpectedDatabase(value="no-users.xml", assertionMode = DatabaseAssertionMode.NON_STRICT)
public void showRegistrationForm_should_render_registration_page_with_empty_form() throws Exception
{
	mockMvc.perform(get("/user/register"))
			.andExpect(status().isOk())
			.andExpect(view().name("user/registrationForm"))
			.andExpect(model().attribute("user", allOf(
					hasProperty("email", isEmptyOrNullString()),
					hasProperty("firstName", isEmptyOrNullString()),
					hasProperty("lastName", isEmptyOrNullString()),
					hasProperty("password", isEmptyOrNullString()),
					hasProperty("confirmPassword", isEmptyOrNullString())
			)));
}
```

En plus de vérifier que l'url affiche bien la vue correspondant au formulaire d'inscription, on vérifie que le modèle alimentant le formulaire est correct.

### Inscription des utilisateurs

On commencer par vérifier les différents cas d'erreurs (attributs manquants ou incorrects). 
Dans ces cas, le modèle doit être mise à jour avec les attributs en erreur et la base de données ne doit pas être modifiée.
Pour voir tous les cas d'erreurs, cf la classe [RegistrationFormSubmitIT](https://raw.githubusercontent.com/jgiovaresco/my-pr/master/src/test/java/integration/tests/RegistrationFormSubmitIT.java)

```java
@Test
@DatabaseSetup("no-users.xml")
@ExpectedDatabase(value="no-users.xml", assertionMode = DatabaseAssertionMode.NON_STRICT)
public void registerUserAccount_should_render_registration_form_with_errors_when_registration_with_empty_form() throws Exception {
	mockMvc.perform(post("/user/register")
							.contentType(MediaType.APPLICATION_FORM_URLENCODED)
							.sessionAttr(RegistrationForm.SESSION_ATTRIBUTE_USER_FORM, new RegistrationForm())
	)
			.andExpect(status().isOk())
			.andExpect(view().name("user/registrationForm"))
			.andExpect(model().attribute(RegistrationForm.MODEL_ATTRIBUTE_USER_FORM, allOf(
					hasProperty(RegistrationForm.FIELD_NAME_EMAIL, isEmptyOrNullString()),
					hasProperty(RegistrationForm.FIELD_NAME_FIRST_NAME, isEmptyOrNullString()),
					hasProperty(RegistrationForm.FIELD_NAME_LAST_NAME, isEmptyOrNullString()),
					hasProperty(RegistrationForm.FIELD_NAME_PASSWORD, isEmptyOrNullString()),
					hasProperty(RegistrationForm.FIELD_NAME_CONFIRM_PASSWORD, isEmptyOrNullString())
			)))
			.andExpect(model().attributeHasFieldErrors(
					RegistrationForm.MODEL_ATTRIBUTE_USER_FORM,
					RegistrationForm.FIELD_NAME_EMAIL,
					RegistrationForm.FIELD_NAME_FIRST_NAME,
					RegistrationForm.FIELD_NAME_LAST_NAME,
					RegistrationForm.FIELD_NAME_PASSWORD,
					RegistrationForm.FIELD_NAME_CONFIRM_PASSWORD
			));
}
```

Le cas nominal est vérifié grâce à la méthode suivante 

```java
@Test
@DatabaseSetup("no-users.xml")
@ExpectedDatabase(value = "register-user-expected.xml", assertionMode = DatabaseAssertionMode.NON_STRICT)
public void registerUserAccount_should_create_new_account_and_render_index_page_when_registration_ok() throws Exception
{
	mockMvc.perform(post("/user/register")
							.contentType(MediaType.APPLICATION_FORM_URLENCODED)
							.param(RegistrationForm.FIELD_NAME_EMAIL, EMAIL)
							.param(RegistrationForm.FIELD_NAME_FIRST_NAME, FIRST_NAME)
							.param(RegistrationForm.FIELD_NAME_LAST_NAME, LAST_NAME)
							.param(RegistrationForm.FIELD_NAME_PASSWORD, PASSWORD)
							.param(RegistrationForm.FIELD_NAME_CONFIRM_PASSWORD, PASSWORD)
							.sessionAttr(RegistrationForm.SESSION_ATTRIBUTE_USER_FORM, new RegistrationForm())
	)
			.andExpect(status().isFound())
			.andExpect(redirectedUrl("/"));
}
```
 
## Le contrôleur

Le contrôleur fournit 2 méthodes : 
 
  * Une méthode qui affiche le formulaire d'inscription
  * Une méthode qui est appelée à la soumission du formulaire pour créer un compte utilisateur.
  
Lors de la soumission, je veux que les données saisies par l'utilisateur soient validées. Pour cela, j'utilise l'api Bean Validation (JSR 303).
Pour déclencher la validation des données, il suffit d'annoter le paramètre de la méthode du contrôleur qui contient les données à valider avec ``@javax.validation.Valid``.

```java
@RequestMapping(value = "/user/register", method = RequestMethod.POST)
public String registerUserAccount(@Valid @ModelAttribute("user") RegistrationForm userAccountData,
								  BindingResult result) throws DuplicateEmailException
{
...
}
```

Pour bien comprendre l'annotation ``@ModelAttribute``, lire cet article de [Javaetmoi](http://javaetmoi.com/2014/10/annotation-sessionattributes-modelattribute-spring-mvc/)
  

## Problème(s) rencontré(s)

Pour tester la validation dans les tests unitaires, il faut annoter la classe de configuration ``fr.mypr.UnitTestWebConfiguration`` avec ``@EnableWebMvc``.

Suite à l'affichage du prénom de l'utilisateur connecté dans la page ``index.html``, le test d'intégration ``IndexPageIT`` est modifié.
Pour simuler l'utilisateur connecté j'utilise un objet ``MyPrUserDetails``  

```java
@Test
public void showIndexPage_asRegisteredUser_should_render_index_page_with_logout_link() throws Exception
{
	UserDetails registeredUser = MyPrUserDetails.builder()
			.username(REGISTERED_USER.getEmail())
			.password(REGISTERED_USER.getPassword())
			.firstName(REGISTERED_USER.getFirstName())
			.build();

	// @formatter:off
	mockMvc.perform(get("/")
			.with(user(registeredUser))
			)
//			.andDo(print())
			.andExpect(status().isOk())
			.andExpect(view().name("index"))
			.andExpect(content().string(containsString("<a href=\"/logout\">Log Out</a>")));
	// @formatter:on
}
```

