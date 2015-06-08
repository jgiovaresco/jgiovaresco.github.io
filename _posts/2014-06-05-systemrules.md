---
layout: post
title:  "Mock les appels à System.exit, System.out, ..."
date:   2014-06-05 18:19:00
categories: [test]
tags: [java, unit tests]
---

Il est difficile de tester du code faisant appel à ``java.lang.System``.
En effet si le code à tester fait un appel à ``System.exit(int)``, la JVM s'arrête avec notre JUnit...

Il existe un ensemble de JUnit rules permettant de tester quand même. Il est donc nécessaire d'utiliser une version de JUnit supérieure à 4.7.

Je ne détaillerai que l'utilisation pour les appels à ``System.exit``, ``System.out`` et ``System.err``. La librairie utilisée permet plus de chose et la documention est plutôt bien fournie 
[Github](http://stefanbirkner.github.com/system-rules)

Un projet exemple se trouve [ici](https://github.com/jgiovaresco/howto-systemrules).

## Dépendance maven

La dépendance maven à utiliser est : 

```xml
<dependency>
 <groupId>com.github.stefanbirkner</groupId>
 <artifactId>system-rules</artifactId>
 <version>1.3.0</version>
 <scope>test</scope>
</dependency>
```

## Mocker System.exit

Pour mocker un appel à ``System.exit``, il suffit de déclarer la Rule :

```java
public class MaClasseTest {
 @Rule
 public final ExpectedSystemExit m_exit = ExpectedSystemExit.none();
...
}
```

Ensuite si on doit s'attendre à un appel à ``System.exit``, dans la méthode de test on utilise 

```java
@Test
public void testExit() {
 m_exit.expectSystemExitWithStatus(5);
 ...
}
```

Dans l'exemple ci-dessus, le test est OK si ``System.exit(5)`` a été appelé.

Cependant avec l'utilisation de cette rule, il n'est plus possible d'effectuer des vérifications (Assert) au retour de la méthode à tester. Cependant la Rule permet de definir 
une callback qui sera appelé à la fin du test nous permettant de faire des vérifications supplémentaire. On la définit avec ``checkAssertionAfterwards()``

```java
@Test
public void testExit() {
 m_exit.expectSystemExitWithStatus(5);
 m_exit.checkAssertionAfterwards(new Assertion() {
  public void checkAssertion() throws Exception {
   Assert.assertTrue(m_aTester.isSortie());
  }
 }); 
 m_aTester.exit(true);
}
```

## Mocker System.out

Pour contrôler le contenu envoyé sur la sortie standart avec ``System.out``, il suffit de déclarer la Rule :

```java
public class MaClasseTest {
 @Rule
 public final StandardOutputStreamLog m_out = new StandardOutputStreamLog();
...
}
```

Ensuite à la fin du test, on peut récupérer le contenu envoyé sur la sortie standart avec 

```java
m_out.getLog()
```

## Mocker System.err

Pour contrôler le contenu envoyé sur la sortie erreur avec ``System.err``, il suffit de déclarer la Rule :

```java
public class MaClasseTest {
 @Rule
 public final StandardErrorStreamLog m_err = new StandardErrorStreamLog();
...
}
```

Ensuite à la fin du test, on peut récupérer le contenu envoyé sur la sortie erreur avec 

```java
m_err.getLog()
```