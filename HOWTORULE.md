# Comment développer des règles pour Sonarqube
Vous utilisez SonarQube et son Java Analyzer pour analyser vos projets, mais il n'existe pas de règles vous permettant de cibler certains besoins spécifiques de votre entreprise ? Ensuite, votre choix logique peut être d'implémenter votre propre ensemble de règles Java personnalisées.

Ce document est une introduction à l'écriture de règles personnalisées pour SonarQube Java Analyzer. Il couvrira tous les principaux concepts d'analyse statique nécessaires pour comprendre et développer des règles efficaces, en s'appuyant sur l'API fournie par SonarSource Analyzer for Java.

## Contenu

* [Mise en route](#getting-started)
   * [En regardant le pompon](#looking-at-the-pom)
* [Écrire une règle](#écrire-une-règle)
   * [Trois fichiers pour forger une règle](#three-files-to-forge-a-rule)
   * [Une spécification pour bien faire les choses](#a-specification-to-make-it-right)
   * [Un fichier de test pour les gouverner tous](#a-test-file-to-rule-them-all)
   * [Une classe de test pour le faire passer](#a-test-class-to-make-it-pass)
   * [Première version : Utilisation des arbres de syntaxe et des bases de l'API](#first-version-using-syntax-trees-and-api-basics)
   * [Deuxième version : Utilisation de l'API sémantique](#second-version-using-semantic-api)
   * [Ce que vous pouvez utiliser et ce que vous ne pouvez pas] (#ce-que-vous-pouvez-utiliser-et-ce-que-vous-ne pouvez pas)
* [Enregistrement de la règle dans le plugin personnalisé](#registering-the-rule-in-the-custom-plugin)
   * [Métadonnées de la règle](#rule-metadata)
   * [Activation de la règle] (#rule-activation)
   * [Registraire des règles](#rule-registrar)
* [Tester un plugin personnalisé](#testing-a-custom-plugin)
   * [Comment définir les paramètres de règle](#how-to-define-rule-parameters)
   * [Comment tester les sources nécessitant des binaires externes](#how-to-test-sources-requiring-external-binaries)
   * [Comment tester l'emplacement précis du problème](#how-to-test-precise-issue-location)
   * [Comment tester la version source dans une règle](#how-to-test-the-source-version-in-a-rule)
* [Références](#références)

## Commencer

Les règles que vous allez développer seront livrées à l'aide d'un plug-in personnalisé dédié, s'appuyant sur l'API **SonarSource Analyzer for Java**. Afin de commencer à travailler efficacement, nous fournissons un modèle de projet maven, que vous remplirez tout en suivant ce tutoriel.

Récupérez le projet de modèle en clonant ce référentiel (https://github.com/SonarSource/sonar-java) puis en important dans votre IDE le sous-module [java-custom-rules-examples](https://github.com /SonarSource/sonar-java/tree/master/docs/java-custom-rules-example).
Ce projet contient déjà des exemples de règles personnalisées. Notre but sera d'ajouter une règle supplémentaire !

### En regardant le POM

Un plugin personnalisé est un projet Maven, et avant de plonger dans le code, il est important de noter quelques lignes pertinentes liées à la configuration de votre plugin personnalisé qui sera bientôt publié. La racine d'un projet Maven est un fichier nommé `pom.xml`.

Dans notre cas, nous en avons 3 :
* `pom.xml` : utilise une version instantanée de l'analyseur Java
* `pom_SQ_8_9_LTS.xml` : fichier `pom` autonome, configuré avec des dépendances correspondant aux exigences SonarQube `8.9 LTS`

Ces 3 `pom`s correspondent à différents cas d'utilisation, selon l'instance de SonarQube que vous ciblerez avec votre plugin de règles personnalisées. Dans ce didacticiel, ** nous n'utiliserons que le fichier nommé `pom_SQ_8_9_LTS.xml` **, car il est complètement indépendant de la version de Java Analyzer, est autonome et ciblera la dernière version de SonarQube.

Commençons par créer le modèle de plugin personnalisé en utilisant la commande suivante :

```
mvn clean install -f pom_SQ_8_9_LTS.xml
```

Notez que vous pouvez également décider de **supprimer** le fichier pom.xml d'origine (**NON RECOMMANDÉ**), puis de renommer `pom_SQ_8_9_LTS.xml` en `pom.xml`. Vous pourrez alors utiliser la commande très simple :

```
mvn clean install
```

En regardant à l'intérieur du `pom`, vous verrez que les deux versions de SonarQube et de Java Analyzer sont codées en dur. En effet, les analyseurs de SonarSource sont directement intégrés dans les différentes versions de SonarQube et sont livrés ensemble. Par exemple, SonarQube `7.9` (ancien LTS) est livré avec la version `6.3.2.22818` de Java Analyzer, tandis que SonarQube `8.9` (LTS) est livré avec une version beaucoup plus récente `6.15.1.26025` de Java Analyzer. Analyseur. **Ces versions ne peuvent pas être modifiées**.

```xml
<properties>
  <sonarqube.version>8.9.0.43852</sonarqube.version>
  <sonarjava.version>6.15.1.26025</sonarjava.version>
  <!-- [...] -->
</properties>
```

D'autres balises telles que `<groupId>`, `<artifactId>`, `<version>`, `<name>` et `<description>` peuvent être librement modifiées.

```xml
  <groupId>org.sonar.samples</groupId>
  <artifactId>java-custom-rules-example</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <name>SonarQube Java :: Documentation :: Custom Rules Example</name>
  <description>Java Custom Rules Example for SonarQube</description>
```

Dans l'extrait de code ci-dessous, il est important de noter que le **point d'entrée du plugin** est fourni en tant que `<pluginClass>` dans la configuration du plugin sonar-packaging-maven, en utilisant le nom complet du classe Java `MyJavaRulesPlugin`.
Si vous refactorisez votre code, renommez ou déplacez la classe étendant `org.sonar.api.SonarPlugin`, vous devrez modifier cette configuration.
C'est aussi la propriété `<sonarQubeMinVersion>` qui garantit la compatibilité avec l'instance SonarQube que vous ciblez.

```xml
<plugin>
  <groupId>org.sonarsource.sonar-packaging-maven-plugin</groupId>
  <artifactId>sonar-packaging-maven-plugin</artifactId>
  <version>1.20.0.405</version>
  <extensions>true</extensions>
  <configuration>
    <pluginKey>java-custom</pluginKey>
    <pluginName>Java Custom Rules</pluginName>
    <pluginClass>org.sonar.samples.java.MyJavaRulesPlugin</pluginClass>
    <sonarLintSupported>true</sonarLintSupported>
    <sonarQubeMinVersion>${sonarqube.version}</sonarQubeMinVersion>
    <requirePlugins>java:${sonarjava.version}</requirePlugins>
  </configuration>
</plugin>
```

## Écrire une règle

Dans cette section, nous allons écrire une règle personnalisée à partir de zéro. Pour ce faire, nous utiliserons une approche [Test Driven Development](https://en.wikipedia.org/wiki/Test-driven_development) (TDD), reposant d'abord sur l'écriture de cas de test, puis sur la mise en œuvre d'une solution.

### Trois fichiers pour forger une règle

Lors de la mise en place d'une règle, il y a toujours un minimum de 3 fichiers distincts à créer :
   1. Un fichier de test, qui contient du code Java utilisé comme données d'entrée pour tester la règle
   1. Une classe de test, qui contient le test unitaire de la règle
   1. Une classe de règles, qui contient l'implémentation de la règle.
  
Pour créer notre première règle personnalisée (généralement appelée "*vérifier*"), commençons par créer ces 3 fichiers dans le projet modèle, comme décrit ci-dessous :

   1. Dans le dossier `/src/test/files`, créez un nouveau fichier vide nommé `MyFirstCustomCheck.java` et copiez-collez le contenu de l'extrait de code suivant.

```java
class MyClass {
}
```

2. Dans le package `org.sonar.samples.java.checks` de `/src/test/java`, créez une nouvelle classe de test appelée `MyFirstCustomCheckTest` et copiez-collez le contenu de l'extrait de code suivant.
```java
package org.sonar.samples.java.checks;
 
import org.junit.jupiter.api.Test;

class MyFirstCustomCheckTest {

  @Test
  void test() {
  }

}
```

3. Dans le package `org.sonar.samples.java.checks` de `/src/main/java`, créez une nouvelle classe appelée `MyFirstCustomCheck` étendant la classe `org.sonar.plugins.java.api.IssuableSubscriptionVisitor` fournie par l'API du plug-in Java. Ensuite, remplacez le contenu de la méthode `nodesToVisit()` par le contenu de l'extrait de code suivant. Ce dossier sera décrit lors de la mise en place de la règle !
```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.Tree.Kind;
import java.util.Collections;
import java.util.List;

@Rule(key = "MyFirstCustomRule")
public class MyFirstCustomCheck extends IssuableSubscriptionVisitor {

  @Override
  public List<Kind> nodesToVisit() {
    return Collections.emptyList();
  }
}

```

>
> :question: **Plus de fichiers...**
>
> Si les 3 fichiers décrits ci-dessus sont toujours la base de l'écriture des règles, il existe des situations où des fichiers supplémentaires peuvent être nécessaires. Par exemple, lorsqu'une règle utilise des paramètres ou si son comportement dépend de la version détectée de Java, plusieurs fichiers de test peuvent être nécessaires. Il est également possible d'utiliser des fichiers externes pour décrire les métadonnées des règles, comme une description au format HTML. Ces situations seront décrites dans d'autres rubriques de cette documentation.
>

### Un cahier des charges pour bien faire les choses

Bien sûr, avant d'aller plus loin, il nous faut un élément clé dans la rédaction de règles : un cahier des charges ! Pour les besoins de l'exercice, considérons la citation suivante d'un célèbre gourou comme étant la spécification de notre règle coutumière, car elle est bien sûr absolument correcte et incontestable.

>
> **Gandalf - Pourquoi programmer quand Magic Rulez (WPWMR, p.42)**
>
> *"Pour une méthode ayant un seul paramètre, les types de sa valeur de retour et de son paramètre ne doivent jamais être les mêmes."*
>

### Un fichier de test pour les gouverner tous

Parce que nous avons choisi une approche TDD, la première chose à faire est d'écrire des exemples du code que notre règle ciblera. Dans ce dossier, nous considérons de nombreux cas que notre règle peut rencontrer lors d'une analyse, et signalons les lignes qui nécessiteront notre implémentation pour soulever des problèmes. L'indicateur à utiliser est un simple commentaire de fin "// non conforme" sur la ligne de code où un problème doit être signalé. Pourquoi *Non conforme* ? Parce que les lignes signalées ne sont pas *conformes* à la règle.

Couvrir tous les cas possibles n'est pas obligatoire, le but de ce fichier est de couvrir toutes les situations qui peuvent être rencontrées lors d'une analyse, mais aussi d'abstraire les détails non pertinents. Par exemple, dans le contexte de notre première règle, le nom de la méthode, le contenu de son corps et le propriétaire de la méthode ne font aucune différence, qu'il s'agisse d'une classe abstraite, d'une classe concrète ou d'une interface. Notez que cet exemple de fichier doit être structurellement correct et que tout le code doit être compilé.

Dans le fichier de test `MyFirstCustomCheck.java` créé précédemment, copiez-collez le code suivant :
```java
class MyClass {
  MyClass(MyClass mc) { }

  int     foo1() { return 0; }
  void    foo2(int value) { }
  int     foo3(int value) { return 0; } // Noncompliant
  Object  foo4(int value) { return null; }
  MyClass foo5(MyClass value) {return null; } // Noncompliant

  int     foo6(int value, String name) { return 0; }
  int     foo7(int ... values) { return 0;}
}
```

Le fichier de test contient maintenant les cas de test suivants :
* **ligne 2 :** Un constructeur, pour différencier le cas d'une méthode ;
* **ligne 4 :** Une méthode sans paramètre (`foo1`);
* **ligne 5 :** Une méthode renvoyant void (`foo2`);
* **ligne 6 :** Une méthode renvoyant le même type que son paramètre (`foo3`), qui sera non conforme ;
* **ligne 7 :** Une méthode avec un seul paramètre, mais un type de retour différent (`foo4`);
* **ligne 8 :** Une autre méthode avec un seul paramètre et même type de retour, mais avec des types non primitifs (`foo5`), donc non conforme également ;
* **ligne 10 :** Une méthode avec plus d'un paramètre (`foo6`);
* **ligne 11 :** Une méthode avec un argument d'arité variable (`foo7`);

### Une classe de test pour le faire passer

Une fois le fichier de test mis à jour, mettons à jour notre classe de test pour l'utiliser, et lions le test à notre règle (pas encore implémentée). Pour ce faire, revenez à notre classe de test `MyFirstCustomCheckTest` et mettez à jour la méthode `test()` comme indiqué dans l'extrait de code suivant (vous devrez peut-être importer la classe `org.sonar.java.checks.verifier.CheckVerifier` ):
```java
  @Test
  void test() {
    CheckVerifier.newVerifier()
      .onFile("src/test/files/MyFirstCustomCheck.java")
      .withCheck(new MyFirstCustomCheck())
      .verifyIssues();
  }
```

Comme vous l'avez sans doute remarqué, cette classe de test contient un seul test dont le but est de vérifier le comportement de la règle que nous allons implémenter. Pour ce faire, il s'appuie sur l'utilisation de la classe `CheckVerifier`, fournie par l'API de test de règles Java Analyzer. Cette classe `CheckVerifier` fournit des méthodes utiles pour valider les implémentations de règles, nous permettant d'abstraire totalement tous les mécanismes liés à l'initialisation de l'analyseur. Notez que lors de la vérification d'une règle, le *vérificateur* collectera les lignes marquées comme étant *Non conforme* et vérifiera que la règle soulève les problèmes attendus et *uniquement* ces problèmes.

Passons maintenant à l'étape suivante de TDD : faites échouer le test !

Pour ce faire, exécutez simplement le test à partir du fichier de test à l'aide de JUnit. Le test doit **échouer** avec le message d'erreur "**Au moins un problème attendu**", comme indiqué dans l'extrait de code ci-dessous. Étant donné que notre vérification n'est pas encore implémentée, aucun problème ne peut encore être soulevé, c'est donc le comportement attendu.

```
java.lang.AssertionError: No issue raised. At least one issue expected
    at org.sonar.java.checks.verifier.InternalCheckVerifier.assertMultipleIssues(InternalCheckVerifier.java:291)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.checkIssues(InternalCheckVerifier.java:231)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.verifyAll(InternalCheckVerifier.java:222)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.verifyIssues(InternalCheckVerifier.java:167)
    at org.sonar.samples.java.checks.MyFirstCustomCheckTest.test(MyFirstCustomCheckTest.java:13)
    ...
```

### Première version : Utilisation des arbres de syntaxe et des bases de l'API

Avant de commencer avec la mise en œuvre de la règle elle-même, vous avez besoin d'un peu de contexte.

Avant d'exécuter une règle, SonarQube Java Analyzer analyse un fichier de code Java donné et produit une structure de données équivalente : l'**arbre de syntaxe**. Chaque construction du langage Java peut être représentée par un type spécifique d'arbre de syntaxe, détaillant chacune de ses particularités. Chacune de ces constructions est associée à un Kind spécifique ainsi qu'à une interface décrivant explicitement toutes ses particularités. Par exemple, le kind associé à la déclaration d'une méthode sera `org.sonar.plugins.java.api.tree.Tree.Kind.METHOD`, et son interface définie par `org.sonar.plugins.java.api. tree.MethodTree`. Tous les types sont répertoriés dans l'énumération [`Kind` de l'API Java Analyzer](https://github.com/SonarSource/sonar-java/blob/6.13.0.25138/java-frontend/src/main/java/ org/sonar/plugins/java/api/tree/Tree.java#L47).

Lors de la création de la classe de règles, nous avons choisi d'implémenter la classe `IssuableSubscriptionVisitor` à partir de l'API. Cette classe, en plus de fournir un tas de méthodes utiles pour soulever des problèmes, **définit également la stratégie qui sera utilisée lors de l'analyse d'un fichier**. Comme son nom l'indique, il est basé sur un mécanisme de souscription, permettant de spécifier sur quel type d'arbre la règle doit réagir. La liste des types de nœuds à couvrir est spécifiée via la méthode `nodesToVisit()`. Dans les étapes précédentes, nous avons modifié l'implémentation de la méthode pour retourner une liste vide, donc ne souscrire à aucun nœud de l'arbre syntaxique.

Il est enfin temps de passer à la mise en œuvre de notre première règle ! Revenez à la classe `MyFirstCustomCheck` et modifiez la liste des Kinds renvoyés par la méthode nodesToVisit(). Étant donné que notre règle cible les déclarations de méthode, nous n'avons qu'à visiter les méthodes. Pour ce faire, assurez-vous simplement que nous renvoyons une liste singleton contenant uniquement `Kind.METHOD` en tant que paramètre de la liste renvoyée, comme indiqué dans l'extrait de code suivant.

```java
@Override
public List<Kind> nodesToVisit() {
  return Collections.singletonList(Kind.METHOD);
}
```

Une fois les nœuds à visiter spécifiés, nous devons implémenter comment la règle réagira lorsqu'elle rencontrera des déclarations de méthode. Pour ce faire, remplacez la méthode `visitNode(Tree tree)`, héritée de `SubscriptionVisitor` via `IssuableSubscriptionVisitor`.

```java
@Override
public void visitNode(Tree tree) {
}
```

Parce que nous avons enregistré la règle pour visiter les nœuds de méthode, nous savons que chaque fois que la méthode est appelée, le paramètre d'arbre sera un `org.sonar.plugins.java.api.tree.MethodTree` (l'arbre d'interface associé au `METHOD ` genre). Dans un premier temps, nous pouvons par conséquent convertir en toute sécurité l'arbre directement dans un MethodTree, comme indiqué ci-dessous. Notez que si nous avions enregistré plusieurs types de nœuds, nous aurions dû tester le type de nœud avant de procéder à la conversion en utilisant la méthode `Tree.is(Kind ... kind)`.

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
}
```

Maintenant, restreignons le champ d'application de la règle en vérifiant que la méthode a un seul paramètre, et soulevons un problème si c'est le cas.

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
  if (method.parameters().size() == 1) {
    reportIssue(method.simpleName(), "Never do that!");
  }
}
```

La méthode `reportIssue(Tree tree, String message)` de `IssuableSubscriptionVisitor` permet de signaler un problème sur un arbre donné avec un message spécifique. Dans ce cas, nous avons choisi de signaler le problème à un endroit précis, qui sera le nom de la méthode.

Maintenant, testons notre implémentation en exécutant à nouveau `MyFirstCustomCheckTest.test()`.

```
java.lang.AssertionError: Unexpected at [5, 7, 11]
    at org.sonar.java.checks.verifier.InternalCheckVerifier.assertMultipleIssues(InternalCheckVerifier.java:303)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.checkIssues(InternalCheckVerifier.java:231)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.verifyAll(InternalCheckVerifier.java:222)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.verifyIssues(InternalCheckVerifier.java:167)
    at org.sonar.samples.java.checks.MyFirstCustomCheckTest.test(MyFirstCustomCheckTest.java:13)
    ...
```

Bien sûr, notre test a de nouveau échoué... Le `CheckVerifier` a signalé que les lignes 5, 7 et 11 soulèvent des problèmes inattendus, comme le montre le stack-trace ci-dessus. En regardant notre fichier de test, il est facile de comprendre que soulever un problème la ligne 5 est faux car le type de retour de la méthode est `void`, la ligne 7 est fausse car `Object` n'est pas le même que `int`, et la ligne 11 est également erronée à cause de la variable *arity* de la méthode. Soulever ces problèmes est cependant correct en fonction de notre implémentation, car nous n'avons pas vérifié les types du paramètre et le type de retour. Pour gérer le type, cependant, nous devrons compter sur plus que ce que nous pouvons réaliser en utilisant uniquement la connaissance de l'arbre de syntaxe. Cette fois, nous devrons utiliser l'API sémantique !

>
> :question : **IssuableSubscriptionVisitor et BaseTreeVisitor**
>
> Pour l'implémentation de cette règle, nous avons choisi d'utiliser un `IssuableSubscriptionVisitor` comme base d'implémentation de notre règle. Ce visiteur offre une approche facile pour écrire des règles simples et rapides, car il nous permet de restreindre l'attention de notre règle à un ensemble donné de Sortes à visiter en y souscrivant. Cependant, cette approche n'est pas toujours la plus optimale. Dans une telle situation, il peut être utile de jeter un œil à un autre visiteur fourni avec l'API : `org.sonar.plugins.java.api.tree.BaseTreeVisitor`. Le `BaseTreeVisitor` contient une méthode `visit()` dédiée à chaque type d'arbre de syntaxe, et est particulièrement utile lorsque la visite d'un fichier doit être affinée.
>
> Dans [règles déjà implémentées dans le plugin Java](https://github.com/SonarSource/sonar-java/tree/5.12.1.17771/java-checks/src/main/java/org/sonar/java/checks) , vous pourrez trouver plusieurs règles en utilisant les deux approches : un "IssuableSubscriptionVisitor" comme point d'entrée, aidé par de simples "BaseTreeVisitor" pour identifier le modèle dans d'autres parties du code.
>

### Deuxième version : Utilisation de l'API sémantique

Jusqu'à présent, notre implémentation de règles ne reposait que sur les données fournies directement par l'arbre de syntaxe résultant de l'analyse du code. Cependant, SonarAnalyzer pour Java fournit beaucoup plus concernant le code analysé, car il construit également un ***modèle sémantique*** du code. Ce modèle sémantique fournit des informations relatives à chaque ***symbole*** manipulé. Pour une méthode, par exemple, l'API sémantique fournira des données utiles telles que le propriétaire d'une méthode, ses usages, les types de ses paramètres et son type de retour, l'exception qu'elle peut lancer, etc. N'hésitez pas à explorer la [sémantique package de l'API](https://github.com/SonarSource/sonar-java/tree/6.13.0.25138/java-frontend/src/main/java/org/sonar/plugins/java/api/semantic) afin pour avoir une idée du type d'informations auxquelles vous aurez accès lors de l'analyse !

Mais maintenant, revenons à notre implémentation et profitons de la sémantique.

Une fois que nous savons que notre méthode a un seul paramètre, commençons par obtenir le symbole de la méthode en utilisant la méthode `symbol()` du `MethodTree`.

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
  if (method.parameters().size() == 1) {
    MethodSymbol symbol = method.symbol();
    reportIssue(method.simpleName(), "Never do that!");
  }
}

```

A partir du symbole, il est alors assez facile de récupérer **le type de son premier paramètre**, ainsi que le **type de retour** (Vous devrez peut-être importer `org.sonar.plugins.java.api.semantic .Symbol.MethodSymbol` et `org.sonar.plugins.java.api.semantic.Type`).

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
  if (method.parameters().size() == 1) {
    Symbol.MethodSymbol symbol = method.symbol();
    Type firstParameterType = symbol.parameterTypes().get(0);
    Type returnType = symbol.returnType().type();
    reportIssue(method.simpleName(), "Never do that!");
  }
}
```

Étant donné que la règle ne devrait soulever un problème que lorsque ces deux types sont identiques, nous testons simplement si le type de retour est le même que le type du premier paramètre à l'aide de la méthode `is(String FullyQualifiedName)`, fournie via le `Type` classe, avant de soulever la question.

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
  if (method.parameters().size() == 1) {
    Symbol.MethodSymbol symbol = method.symbol();
    Type firstParameterType = symbol.parameterTypes().get(0);
    Type returnType = symbol.returnType().type();
    if (returnType.is(firstParameterType.fullyQualifiedName())) {
      reportIssue(method.simpleName(), "Never do that!");
    }
  }
}
```

Maintenant, **exécutez à nouveau la classe test**.

Test réussi? Si ce n'est pas le cas, vérifiez si vous avez manqué une étape.

Si c'est passé...

>
> :tada: **Félicitations !** :confetti_ball:
>
> [*Vous avez implémenté votre première règle personnalisée pour SonarQube Java Analyzer !*](resources/success.jpg)
>

### Ce que vous pouvez utiliser et ce que vous ne pouvez pas

Lors de l'écriture de règles Java personnalisées, vous ne pouvez utiliser que les classes du package [`org.sonar.plugins.java.api`](https://github.com/SonarSource/sonar-java/tree/6.13.0.25138/java-frontend /src/main/java/org/sonar/plugins/java/api).

Lorsque vous parcourez les plus de 600 règles existantes de SonarSource Analyzer for Java, vous remarquerez parfois l'utilisation d'autres classes d'utilitaires, ne faisant pas partie de l'API. Bien que ces classes puissent parfois être extrêmement utiles dans votre contexte, **ces classes ne sont pas disponibles au moment de l'exécution** pour les plug-ins de règles personnalisées. Cela signifie que, même si vos tests unitaires vont toujours réussir lors de la construction de votre plugin, vos règles feront très probablement planter l'analyse au moment de l'analyse **.

Notez que nous sommes toujours ouverts à la discussion, alors n'hésitez pas à nous contacter et à participer aux fils de discussion, via notre [forum communautaire](https://community.sonarsource.com/), pour suggérer des fonctionnalités et des améliorations d'API !

## Enregistrement de la règle dans le plugin personnalisé

OK, vous êtes probablement assez satisfait à ce stade, car notre première règle fonctionne comme prévu... Cependant, nous n'avons pas encore vraiment terminé. Avant de jouer notre règle contre tout projet réel, nous devons finaliser sa création au sein du plugin personnalisé, en l'enregistrant.

### Métadonnées de règle
La première chose à faire est de fournir à notre règle toutes les métadonnées qui nous permettront de l'enregistrer correctement dans la plateforme SonarQube.
Il existe deux façons d'ajouter des métadonnées à votre règle : l'annotation et la documentation statique.
Alors que l'annotation fournit un moyen pratique de documenter la règle, la documentation statique offre la possibilité d'obtenir des informations plus riches.
Incidemment, la documentation statique est également la manière dont les règles de l'analyseur sonar-java sont décrites.

Pour fournir des métadonnées pour votre règle, vous devez créer un fichier HTML, dans lequel vous pouvez fournir une description textuelle étendue de la règle, et un fichier JSON, avec les métadonnées réelles.
Dans le cas de `MyFirstCustomRule`, vous vous dirigerez vers le dossier `src/main/resources/org/sonar/l10n/java/rules/java/` pour créer `MyFirstCustomRule.html` et `MyCustomRule.json`.

Nous devons d'abord remplir le fichier HTML avec des informations qui aideront les développeurs à résoudre le problème.

```html
<p>For a method having a single parameter, the types of its return value and its parameter should never be the same.</p>

<h2>Noncompliant Code Example</h2>
<pre>
class MyClass {
  int doSomething(int a) { // Noncompliant
    return 42;
  }
}
</pre>

<h2>Compliant Solution</h2>
<pre>
class MyClass {
  int doSomething() { // Compliant
    return 42;
  }
  long doSomething(int a) { // Compliant
    return 42L;
  }
}
</pre>
```

Nous pouvons maintenant ajouter des métadonnées à `src/main/resources/org/sonar/l10n/java/rules/java/MyFirstCustomRule.json` :
```json
{
  "title": "Return type and parameter of a method should not be the same",
  "type": "Bug",
  "status": "ready",
  "tags": [
    "bugs",
    "gandalf",
    "magic"
  ],
  "defaultSeverity": "Critical"
}
```

Avec cet exemple, nous avons un "titre" concis mais descriptif pour notre règle, le "type" de problème qu'il met en évidence, son "état" (prêt ou obsolète), les "balises" qui devraient le faire apparaître dans une recherche et le « gravité » du problème.

### Activation de la règle
La deuxième chose à faire est d'activer la règle dans le plugin. Pour ce faire, ouvrez la classe `RulesList` (`org.sonar.samples.java.RulesList`). Dans cette classe, vous remarquerez les méthodes `getJavaChecks()` et `getJavaTestChecks()`. Ces méthodes sont utilisées pour enregistrer nos règles avec la règle du plugin Java. Notez que les règles enregistrées dans `getJavaChecks()` ne seront lues que sur les fichiers source, tandis que les règles enregistrées dans `getJavaTestChecks()` ne seront lues que sur les fichiers de test. Pour enregistrer la règle, ajoutez simplement la classe de règles au générateur de liste, comme dans l'extrait de code suivant :
```java
public static List<Class<? extends JavaCheck>> getJavaChecks() {
  return Collections.unmodifiableList(Arrays.asList(
      // other rules...
      MyFirstCustomCheck.class
    ));
}

```

### Registraire des règles

Étant donné que vos règles reposent sur l'API SonarSource Analyzer for Java, vous devez également indiquer au plug-in Java parent que certaines nouvelles règles doivent être récupérées. Si vous utilisez le plugin personnalisé de modèle comme base de ce tutoriel, vous devriez déjà avoir tout fait, mais n'hésitez pas à jeter un œil à la classe `MyJavaFileCheckRegistrar.java`, qui relie les points. Enfin, assurez-vous que cette classe de registre est également correctement ajoutée en tant qu'extension pour votre plugin personnalisé, en l'ajoutant à votre classe de définition de plugin (`MyJavaRulesPlugin.java`).
```java
/**
 * Provide the "checks" (implementations of rules) classes that are going be executed during
 * source code analysis.
 *
 * This class is a batch extension by implementing the {@link org.sonar.plugins.java.api.CheckRegistrar} interface.
 */
@SonarLintSide
public class MyJavaFileCheckRegistrar implements CheckRegistrar {
 
  /**
   * Register the classes that will be used to instantiate checks during analysis.
   */
  @Override
  public void register(RegistrarContext registrarContext) {
    // Call to registerClassesForRepository to associate the classes with the correct repository key
    registrarContext.registerClassesForRepository(MyJavaRulesDefinition.REPOSITORY_KEY, checkClasses(), testCheckClasses());
  }
 
 
  /**
   * Lists all the main checks provided by the plugin
   */
  public static List<Class<? extends JavaCheck>> checkClasses() {
    return RulesList.getJavaChecks();
  }
 
  /**
   * Lists all the test checks provided by the plugin
   */
  public static List<Class<? extends JavaCheck>> testCheckClasses() {
    return RulesList.getJavaTestChecks();
  }
 
}
```

Maintenant, parce que nous avons ajouté une nouvelle règle, nous devons également mettre à jour nos tests pour nous assurer qu'elle est prise en compte. Pour ce faire, accédez à sa classe de test correspondante, nommée "MyJavaFileCheckRegistrarTest", et mettez à jour le nombre de règles attendu de 8 à 9.

```java

class MyJavaFileCheckRegistrarTest {

  @Test
  void checkNumberRules() {
    CheckRegistrar.RegistrarContext context = new CheckRegistrar.RegistrarContext();

    MyJavaFileCheckRegistrar registrar = new MyJavaFileCheckRegistrar();
    registrar.register(context);

    assertThat(context.checkClasses()).hasSize(8); // change it to 9, we added a new one!
    assertThat(context.testCheckClasses()).isEmpty();
  }
}
```

### Référentiel de règles

Avec les actions entreprises ci-dessus, votre règle est activée, enregistrée et devrait être prête à être testée.
Mais avant cela, vous souhaiterez peut-être personnaliser le nom du référentiel auquel appartient votre règle.

La clé et le nom de ce référentiel sont définis dans `MyJavaRulesDefinition.java` et peuvent être personnalisés en fonction de vos besoins.
```java
public class MyJavaRulesDefinition implements RulesDefinition {
  // ...
  public static final String REPOSITORY_KEY = "fellowship-inc";

  public static final String REPOSITORY_NAME = "The Fellowship's custom rules";
  // ...
}
```

## Tester un plugin personnalisé

>
> :exclamation: **Prérequis**
>
> Pour ce chapitre, vous aurez besoin d'une instance locale de SonarQube. Si vous n'avez pas de plateforme SonarQube installée sur votre machine, il est maintenant temps de télécharger sa dernière version depuis [ICI](https://www.sonarqube.org/downloads/) !
>

À ce stade, nous avons terminé la mise en œuvre d'une première règle personnalisée et l'avons enregistrée dans le plug-in personnalisé. La dernière étape restante est de le tester directement avec la plateforme SonarQube et d'essayer d'analyser un projet !

Commencez par construire le projet en utilisant maven. Notez qu'ici nous utilisons le fichier `pom` autonome ciblant SonarQube `8.9` LTS. Si vous l'avez renommé en `pom.xml`, supprimez la partie `-f pom_SQ_8_9_LTS.xml` de la commande suivante :
```
$ pwd
/home/gandalf/workspace/sonar-java/docs/java-custom-rules-example
  
$ mvn clean install -f pom_SQ_8_9_LTS.xml
[INFO] Scanning for projects...
[INFO]                                                                        
[INFO] ------------------------------------------------------------------------
[INFO] Building SonarQube Java :: Documentation :: Custom Rules Example 1.0.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
  
...
 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 8.762 s
[INFO] Finished at: 2021-03-02T12:17:28+01:00
[INFO] ------------------------------------------------------------------------
```

Ensuite, récupérez le fichier jar `java-custom-rules-example-1.0.0-SNAPSHOT.jar` du dossier `target` du projet et déplacez-le dans le dossier extensions de votre instance SonarQube, qui sera situé à `$SONAR_HOME/extensions/plugins`.

>
> :exclamation: **Version compatible avec le plug-in Java SonarQube**
>
> Avant d'aller plus loin, assurez-vous d'avoir la version adéquate du SonarQube Java Analyzer avec votre instance SonarQube. La dépendance sur Java Analyzer de notre plugin personnalisé est définie dans son `pom`, comme vu dans le premier chapitre de ce tutoriel. Nous fournissons par conséquent deux fichiers `pom` distincts mappant à la fois la version précédente LTS `7.9` de SonarQube, ainsi que la dernière version LTS, version `8.9`.
>
> * Si vous utilisez un SonarQube `8.9` et que vous avez déjà mis à jour le dernier LTS, vous n'aurez plus la possibilité de mettre à jour l'analyseur Java de manière indépendante. Par conséquent, utilisez le fichier `pom_SQ_8_9_LTS.xml` pour construire le projet.
>

Maintenant, (re-)démarrez votre instance SonarQube, connectez-vous en tant qu'administrateur et accédez à l'onglet ***Règles***.

À partir de là, sous la section langue, sélectionnez "** Java **", puis " ** Les règles personnalisées de la communauté ** " (ou " ** MyCompany Custom Repository ** " si vous ne l'avez pas modifié) sous le référentiel section. Votre règle devrait maintenant être visible (avec tous les autres exemples de règles).

![Règles sélectionnées](resources/rules_selected.png)

Une fois activé (vous ne savez pas comment ? Voir [quality-profiles](https://docs.sonarqube.org/latest/instance-administration/quality-profiles/)), il ne reste plus qu'à analyser l'un de vos projets !

Lorsque vous rencontrez une méthode renvoyant le même type que son paramètre, le problème soulèvera désormais un problème, comme le montre l'image suivante :

![Problèmes](ressources/problèmes.png)

### Comment définir les paramètres des règles

Vous devez ajouter un `@RuleProperty` à votre règle.

Vérifiez cet exemple : [SecurityAnnotationMandatoryRule.java](https://github.com/SonarSource/sonar-java/blob/master/docs/java-custom-rules-example/src/main/java/org/sonar/samples/ java/checks/SecurityAnnotationMandatoryRule.java)

### Comment tester les sources nécessitant des binaires externes

Dans le `pom.xml`, définissez dans la partie `Maven Dependency Plugin` tous les JAR dont vous avez besoin pour exécuter vos tests unitaires. Par exemple, si votre exemple de code utilisé dans vos tests unitaires dépend de Spring, ajoutez-le ici.

Voir : [pom.xml#L137-L197](./java-custom-rules-example/pom_SQ_8_9_LTS.xml#L137-L197)

### Comment tester l'emplacement précis du problème

Vous pouvez soulever un problème sur une ligne donnée, mais vous pouvez également le soulever sur un jeton spécifique. Pour cette raison, vous souhaiterez peut-être spécifier, dans votre exemple de code utilisé par vos tests unitaires, l'emplacement exact, c'est-à-dire entre 2 colonnes spécifiques, où vous vous attendez à ce que le problème soit soulevé.

Ceci peut être réalisé en utilisant les mots-clés spéciaux `sc` (start-column) et `ec` (end-column) dans le commentaire `// Noncompliant`. Dans l'exemple suivant, nous nous attendons à ce que le problème soit soulevé entre les colonnes 27 et 32 (c'est-à-dire exactement sur le type de variable "Commande") :
```java
public String updateOrder(Order order) { // Noncompliant [[sc=27;ec=32]] {{Don't use Order here because it's an @Entity}}
```

### Comment tester la version source dans une règle

À partir de **Java Plugin API 3.7** (octobre 2015), la version source Java est accessible directement lors de l'écriture de règles personnalisées. Ceci peut être réalisé en appelant simplement la méthode `getJavaVersion()` depuis le contexte. Notez que la méthode renverra null uniquement lorsque la propriété n'est pas définie. De même, il est possible de spécifier au vérificateur une version de Java à considérer comme exécutable, en appelant la méthode `verify(String filename, JavaFileScanner check, int javaVersion)`.
```java
@Beta
public interface JavaFileScannerContext {

  // ...
  
  @Nullable
  Integer getJavaVersion();
  
}
```

## References

* [Analysis of Java code documentation](https://docs.sonarqube.org/latest/analysis/languages/java/)
* [SonarQube Platform](http://www.sonarqube.org/)
* [SonarSource Code Quality and Security for Java Github Repository](https://github.com/SonarSource/sonar-java)
* [SonarQube Java Custom-Rules Example](https://github.com/SonarSource/sonar-java/tree/master/docs/java-custom-rules-example)
