---
layout: post
title:  "My-PR : Une application pour crossfiteur"
date:   2015-06-14 11:13:00
categories: [my-pr]
tags: [java, spring]
---

Je commence une série d'articles consacrée au développement d'une application avec [Spring Boot](http://projects.spring.io/spring-boot/). 
Dans ces articles, je présenterai les étapes de développement de l'application jusqu'à son déploiement.

Cette application a pour but premier de me permettre de jouer avec Spring et de pratiquer quelques bonnes pratiques (Clean Code, TDD, ...).
Le développement se fera en itérations et chaque étape sera décrite.

# Etape 1 : Application basique
 
Le done de cette étape est une application affichant une page index contenant un message de bienvenue.

Les sources de cette étape se trouvent sur [GitHub](https://github.com/jgiovaresco/my-pr/tree/step1) 

## Création de l'application

Pour créer l'application, j'utilise l'outil [Spring Initializr](https://start.spring.io) intégré à Intellij.

### Project metadata

 * Type : Gradle project
 * Packaging : jar
 * Spring Boot Version : 1.3.0 M1
 
Pour cette application, je vais utiliser [Gradle](https://gradle.org/) pour améliorer ma pratique afin de pouvoir ensuite l'utiliser au quotidien. 

Le packaging sera un jar exécutable pour faciliter le déploiement.

J'utilise [Spring Boot 1.3.0 M1](https://spring.io/blog/2015/06/12/spring-boot-1-3-0-m1-available-now) dont la release devrait sortir en Septembre. 
Cette version utilise [Spring Framework 4.2.0.RC1](https://spring.io/blog/2015/05/26/spring-framework-4-2-goes-rc1).

### Project dependencies

Pour cette 1ère étape, les dépendances Spring seront 

 * Web
 * Thymeleaf 

Ces seules dépendances permettent de répondre aux besoins de l'étape 1.
 
## Création du test d'acceptation.

Grâce à Spring, on peut facilement écrire un test permettant de valider l'exigence.

```java
	@Test
	public void should_show_index_page() throws Exception
	{
		mockMvc.perform(get("/"))
//				.andDo(print())
				.andExpect(status().isOk())
				.andExpect(view().name("index"))
				.andExpect(forwardedUrl("templates/index.html"));
	}
```

Nous avons 2 choix possibles pour la déclarer la configuration de ce test. 
 
  1. Utiliser la configuration automatique de Spring Boot en annotant la classe de test avec ``@ContextConfiguration(classes = MyPrApplication.class)``
  2. Configurer manuellement le view resolver
  
Le 1er choix est plus rapide à mettre en place, l'inconvénient est que le test sera plus long car Spring Boot va configurer toute l'application (cache des templates, ...)
Nous utiliserons cette méthode pour faire des tests d'intégration.

Du coup, nous allons plutôt configurer le ViewResolver manuellement, grâce à l'inner-class suivante

```java
	@Configuration
	@ComponentScan(basePackageClasses = IndexController.class)
	public static class TestConfiguration
	{
		@Bean
		public ViewResolver viewResolver()
		{
			InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
			viewResolver.setPrefix("templates/");
			viewResolver.setSuffix(".html");
			return viewResolver;
		}
	}
```
 
## Création du contrôleur et du template

### Le contrôleur

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class IndexController
{
	@RequestMapping("/")
	public String showFirstPage(Model p_model)
	{
		p_model.addAttribute("name", "My PR");
		return "index";
	}
}
```

La méthode ``showFirstPage`` sera appelée lors d'une requête sur ``/``. Celle-ci alimente le model avec un attribut ``name`` et renvoie le 
nom du template à utiliser.

### Le template

```xml
<!DOCTYPE HTML>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
	<title>My PR</title>
	<meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
</head>
<body>
	<h1>Welcome to <span th:text="${name}">my-pr</span></h1>

	<p>This application allows you to register yours Personal Records.</p>
</body>
</html>
```

Le template se trouve dans le fichier ``index.html`` placé dans le répertoire ``src/main/resources/templates``.

## Intégration continue

L'application étant sur [Github](https://github.com/jgiovaresco), je peux utiliser [TravisCI](https://travis-ci.org/) comme plateforme d'intégration continue.

Après avoir lié mon compte GitHub avec Travis, il suffit d'ajouter un fichier ``.travis.yml`` à la racine du projet pour déclencher les builds à chaque push.
Dans ce fichier, on déclare simplement qu'il s'agit d'un projet Java et qu'il faut utiliser le JDK 8.

```yaml
language: java
jdk:
  - oraclejdk8
```

## Problème(s) rencontré(s)

Le seul problème que j'ai rencontré ici vient de l'utilisation du plugin [Infinitest](https://infinitest.github.io/). Cet outil permet de lancer les tests à chaque modification du code.
L'avantage de cet outil est qu'il relance uniquement les tests impactés par la modification.  

Cependant, le test écrit dans cette étape ne fonctionne pas avec Infinitest. On obtient l'erreur suivante
 
```
fr.mypr.controller.IndexControllerTest:67 - NestedServletException(Request processing failed; nested exception is org.thymeleaf.exceptions.TemplateInputException: Error resolving template "index", template might not exist or might not be accessible by any of the configured Template Resolvers)
```

Or ce test fonctionne s'il est lancé directement avec Intellij ou avec Gradle.
Pour éviter ce problème, le test n'utilise pas Thymeleaf mais un ViewResolver simple.

 