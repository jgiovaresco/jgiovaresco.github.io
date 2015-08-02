---
layout: post
title:  "My-PR : Utilisation de Liquibase"
date:   2015-08-02 18:23:00
categories: [my-pr]
tags: [java, spring, liquibase]
---


Dans cet article, je présente la mise en place de [Liquibase](http://www.liquibase.org/) pour gérer le schéma de la Base de Données de l'application.

Les sources de cette étape se trouvent sur [GitHub](https://github.com/jgiovaresco/my-pr/tree/step6) 

## Présentation de Liquibase

[Liquibase](http://www.liquibase.org/) est un outil permettant de gérer le schéma de notre Base de Données. Son intérêt vient du fait qu'il gère la migration du schéma de manière incrémental.
Cet outil agit comme un système de gestion de version pour le schéma de notre Base de Données.

L'outil est organisé en ``changesets``. Ces changesets sont identifiés par l'id, l'auteur et son checksum. Par défaut le changeset ne sera joué qu'une seule fois et il est immutable.
Grâce au checksum, Liquibase détecte que le changeset appliqué a été modifié et lèvera une erreur.  

Les changesets peuvent être écrit en XML, YAML, JSON et SQL. En plus de ces formats, la communauté fournit des formats supplémentaires comme [Groovy Liquibase](https://github.com/liquibase/liquibase-groovy-dsl).
Il est même possible de décrire un changeset en [Java](http://jeanchristophegay.com/liquibase-refactoring-change-java/).
 
Exemple de changeset en XML :

```xml
<changeSet id="0.0.1" author="jgi" context="db">
	<createTable tableName="USER_ACCOUNT">
		<column name="ID" type="bigint" autoIncrement="true">
			<constraints primaryKey="true" nullable="false"/>
		</column>
		<column name="EMAIL" type="varchar(100)">
			<constraints nullable="false" />
		</column>
		<column name="PASSWORD" type="varchar(100)">
			<constraints nullable="false" />
		</column>
		<column name="FIRST_NAME" type="varchar(50)">
			<constraints nullable="false" />
		</column>
		<column name="LAST_NAME" type="varchar(50)">
			<constraints nullable="false" />
		</column>
	</createTable>
</changeSet>
```

Pour fonctionner, Liquibase s'appuie sur une table ``DATABASECHANGELOG`` dans laquelle il stocke l'historique des modifications.

Il est possible d'exécuter Liquibase avec

* la ligne de commande
* Ant, Maven ou Groovy
* Spring

## Mise en place dans My-PR

La création / mise à jour de la Base de Données se fait à 2 moments : 

* avant d'exécuter les tests d'intégration (création from scratch à chaque fois)
* lors de la mise en production (création la 1ère fois puis mise à jour)

Je vais donc configurer Liquibase pour qu'il soit appelé

* par Spring pour les tests d'intégration.
* par Gradle pour gérer la Base de Données des environnements de déploiement.

Les fichiers décrivant les changesets à appliquer seront embarqués dans l'application. De plus ils sont organisés de la façon suivante : 
 
* un fichier regroupant tous les autres : **db.changelog.xml**
* un fichier décrivant les mises à jour successives : **db-0.0.1.xml, db-0.0.2.xml...**

### Appel depuis Spring

Pour appeler Liquibase depuis Spring, il suffit de déclarer un Bean ``liquibase.integration.spring.SpringLiquibase``. Pour cela, je vais créer une classe de Configuration spécifique aux tests d'intégration

```java
@Profile({"integrationTest"})
@Import({MyPrApplication.class})
@Configuration
public class IntegrationTestConfig
{
	@Bean
	public SpringLiquibase liquibase(DataSource dataSource)
	{
		SpringLiquibase springLiquibase = new SpringLiquibase();
		springLiquibase.setDataSource(dataSource);
		springLiquibase.setChangeLog("classpath:liquibase/db.changelog.xml");
		return springLiquibase;
	}
}
```

### Appel depuis Gradle

J'ai voulu extraire dans un fichier particulier la gestion de Liquibase. Cependant je ne suis pas arrivé à résultat concluant dû à une méconnaissance de Gradle. 

La configuration du plugin Liquibase se trouve dans le fichier ``database.gradle`` et elle est importée dans ``build.gradle``. 
Cependant il est nécessaire de déclarer des dépendances supplémentaires dans ``build.gradle``.

Le fichier ``database.gradle`` gère uniquement une base de donnée H2 locale pour le moment :

```groovy
apply plugin: 'liquibase'

def liquibaseChangeLogFile = 'src/main/resources/liquibase/db.changelog.xml'
def liquibaseContexts = 'db'

liquibase {
    activities {
        dev {
            changeLogFile liquibaseChangeLogFile
            contexts liquibaseContexts
            url 'jdbc:h2:file:/home/julien/DEV/perso/my-pr/db/my-pr'
            username 'sa'
            password 'sa'
        }
    }
}
```

Dans ``build.gradle``

```groovy
buildscript {
    ...
    dependencies {
        ...
        classpath 'org.liquibase:liquibase-gradle-plugin:1.1.0'
        classpath 'com.h2database:h2:1.3.160'
    }
}
...
apply from: 'database.gradle'
```

Pour exécuter une migration, on appelle la commande ``gradle update -Pdev``

```
$ gradle update -Pdev                
:update
liquibase-plugin: Running the 'dev' activity...
INFO 8/2/15 6:12 PM: liquibase: Successfully acquired change log lock
INFO 8/2/15 6:12 PM: liquibase: Creating database history table with name: PUBLIC.DATABASECHANGELOG
INFO 8/2/15 6:12 PM: liquibase: Reading from PUBLIC.DATABASECHANGELOG
INFO 8/2/15 6:12 PM: liquibase: src/main/resources/liquibase/db.changelog.xml: src/main/resources/liquibase/db-0.0.1.xml::0.0.1::jgi: Table USER_ACCOUNT created
INFO 8/2/15 6:12 PM: liquibase: src/main/resources/liquibase/db.changelog.xml: src/main/resources/liquibase/db-0.0.1.xml::0.0.1::jgi: ChangeSet src/main/resources/liquibase/db-0.0.1.xml::0.0.1::jgi ran successfully in 4ms
INFO 8/2/15 6:12 PM: liquibase: src/main/resources/liquibase/db.changelog.xml: src/main/resources/liquibase/db-0.0.2.xml::0.0.1::jgi: Table EXERCISES created
INFO 8/2/15 6:12 PM: liquibase: src/main/resources/liquibase/db.changelog.xml: src/main/resources/liquibase/db-0.0.2.xml::0.0.1::jgi: Table PERSONAL_RECORDS created
INFO 8/2/15 6:12 PM: liquibase: src/main/resources/liquibase/db.changelog.xml: src/main/resources/liquibase/db-0.0.2.xml::0.0.1::jgi: ChangeSet src/main/resources/liquibase/db-0.0.2.xml::0.0.1::jgi ran successfully in 5ms
INFO 8/2/15 6:12 PM: liquibase: Successfully released change log lock
Liquibase Update Successful

BUILD SUCCESSFUL

Total time: 13.592 secs
```