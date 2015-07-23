---
layout: post
title:  "Déclaration d'un base de donnée embarquée avec Spring et JavaConfig"
date:   2015-07-23 19:08:00
categories: [test]
tags: [java, spring]
---

Pour les tests unitaires des classes qui interagissent avec une base de donnée, on peut utiliser une base de données en mémoire type [H2]() ou [HSQLDB]().

Grâce à Spring JDBC, on peut faire construire une base très rapidement 
 
```java
	@Bean
	public DataSource getDataSource() throws Exception
	{
		return new EmbeddedDatabaseBuilder()
				.setType(EmbeddedDatabaseType.H2)
				.addScript("classpath:/schema.sql")
				.build();
	}
```

La base sera créée et initialisée avec le(s) script(s) passés au builder. Il est possible de définir le type de base (H2, HSQL ou DERBY).

Attention : lorsqu'une base de donnée en mémoire est déclarée dans une classe de Configuration, elle peut surcharger la déclaration Spring Boot pour des tests d'intégration.
Pour éviter toute erreur, j'utilise un profil particulier pour la classe Configuration définissant la base de donnée pour mes tests unitaires. 