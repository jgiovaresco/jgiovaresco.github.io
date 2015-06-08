---
layout: post
title:  "Utilisation de PowerMock pour mocker méthodes static, final ou des constructeurs"
date:   2014-06-05 18:07:00
categories: [test]
tags: [java, mockito, powermock, unit tests]
---

Les frameworks de Mocks comme EasyMock ou Mockito sont incapables de mocker les appels à des méthodes static ou final ou les appels aux constructeurs.
Heureusement, [PowerMock](https://code.google.com/p/powermock/) est là pour pallier aux limites de ces frameworks. Je ne détaillerai ici que l'utilisation de PowerMock avec EasyMock.


## Dépendances maven

Les dépendances maven pour utiliser PowerMock sont :

```xml
<dependency>
 <groupId>org.powermock</groupId>
 <artifactId>powermock-module-junit4</artifactId>
 <version>1.4.12</version>
 <scope>test</scope>
</dependency>
<dependency>
 <groupId>org.powermock</groupId>
 <artifactId>powermock-api-easymock</artifactId>
 <version>1.4.12</version>
 <scope>test</scope>
</dependency>
```

## Utilisation 

1) Il est nécessaire d'annoter la classe de test avec 

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest({ MaClasseStatic.class })
```

Les valeurs de l'annotation ``@PrepareForTest`` contiennent la liste des classes qui seront "préparées" par PowerMock. Ce seront donc 

* les classes contenant les méthodes static ou final
* les classes faisant appel aux constructeurs à mocker

2) Ensuite on configure les mocks

Pour mocker une méthode static

```java
mockStatic(MaClasseStatic.class);
expect(MaClasseStatic.methode1("abcd")).andReturn("azerty");
replay(MaClasseStatic.class);
```

Pour mocker un constructeur

```java
File mockFile = createMockAndExpectNew(File.class, "monFichier");
expect(mockFile.getAbsolutePath()).andReturn("monChemin");
PowerMock.replay(mockFile, File.class);
```

## Utilisation de PowerMockRule

Depuis JUnit 4.7, le framework permet l'utilisation de Rule. Ceci nous permet de nous passer de l'annotation ``@RunWith``. En effet une classe de test ne peut accepter qu'une seule annotation ``@RunWith``.
PowerMock fournit une Rule remplacant l'annotation ``@RunWith``

Il est nécessaire d'ajouter les dépendances maven : 

```xml
<dependency>
 <groupId>org.powermock</groupId>
 <artifactId>powermock-module-junit4-rule</artifactId>
 <version>1.4.12</version>
 <scope>test</scope>
</dependency>
<dependency>
 <groupId>org.powermock</groupId>
 <artifactId>powermock-classloading-xstream</artifactId>
 <version>1.4.12</version>
 <scope>test</scope>
</dependency>
```

Dans le code 

```java
@PrepareForTest({ MaClasseStatic.class })
public class MockStatic_RuleTest {
 
 @Rule
 public PowerMockRule rule = new PowerMockRule();
    
 private MaClasse1 m_aTester;
...
}
```

Pour plus de détail, le site du projet est plutôt bien fournit en doc : [PowerMock](https://code.google.com/p/powermock/)

Un projet exemple se trouve [ici](https://github.com/jgiovaresco/howto-powermock).