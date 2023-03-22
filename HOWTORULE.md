# Comment écrire une règle pour un plugin Sonarqube
Ce guide a comme objectif de vous faire gagner du temps :
* Dans l’initialisation de votre environnement de développement,
* Dans la compréhension de l’écriture de règles pour sonarqube,
* Dans la réalisation des tests de ces règles pour les valider.

Nous allons aborder toutes ces étapes et partir d’une règle déjà écrite pour vous expliquer comment faire.

Cette règle va concerner le plugin d’analyse de code Python. Nous allons donc écrire un fichier de test en Python et du code en Java pour l’analyse de ce fichier de test.

Pour ce faire, nous utiliserons une approche de développement piloté par les tests (TDD), reposant d'abord sur l'écriture de cas de test, suivie de l'implémentation d'une solution.

# Initialisation de votre environnement de développement
Le projet ecoCode a rédigé des explications sur comment initialiser sont environnement de développement.

Les informations se trouvent sur ce lien [Starter-pack]( https://github.com/green-code-initiative/ecoCode-common/blob/main/doc/starter-pack.md)

Le projet met aussi à votre disposition des explications et des outils pour faire fonctionner un sonarqube local pour tester vos règles.

Les informations se trouvent sur ce lien [Installation]( https://github.com/green-code-initiative/ecoCode-common/blob/main/doc/INSTALL.md)

# Comment écrire une règle Sonarqube
## Quels sont les fichiers à créer pour coder une règle
Les fichiers à construire pour écrire une règle sont :
* Un fichier de test :
    * Qui contient le test unitaire de la règle, c’est donc du code en Python
    * Il se place dans ecoCodeCodespaces/python-plugin/src/test/resources/checks
* Une classe de test :
    * Qui contient du code Java utilisé comme données d'entrée pour tester la règle
    * Il se place dans ecoCodeCodespaces/python-plugin/src/test/java/fr/greencodeinitiative/python/checks
* Une classe de règles :
    * Qui contient l'implémentation de la règle
    * Il se place dans ecoCodeCodespaces/python-plugin/src/main/java/fr/greencodeinitiative/python/checks
* Un fichier de description de la règle, au format json :
    * Qui contient les informations utilisées par Sonarqube pour identifier la règle
    * Il se place dans ecoCodeCodespaces/python-plugin/src/main/resources/fr/greencodeinitiative/l10n/python/rules/python
* Un fichier de description des consignes aux développeurs si la règles est activée, au format html :
    * Qui contient le texte à afficher au développeur quand la règle est activée par Sonarqube,
    * Il se place dans ecoCodeCodespaces/python-plugin/src/main/resources/fr/greencodeinitiative/l10n/python/rules/python

## Avant de commencer, ou trouver des règles à coder
Des règles sont disponibles sur les liens suivants :
* [ecoCode Standard - Candidate Rules]( https://github.com/orgs/green-code-initiative/projects/1)
* [ecoCode Mobile - Candidate Rules](https://github.com/orgs/green-code-initiative/projects/4/views/1)

Ces règles ont été identifiées par des contributeurs au projet. L’objectif du hackathon est d’identifier de nouvelles règles et surtout de coder en java dans les différents plugins les règles présentent dans les deux projets.

## Alors, comment coder une règle
Nous allons prendre et décortiquer une règle déjà écrite, l’objectif est de vous faire gagner du temps en vous expliquant quoi coder.

### Commençons par prendre une règle
Première étape de la démarche, choisir une règle à coder dans un des deux projets qui ont été présentés ci-dessus. Il faudra bien penser à changer le statut du ticket dans le projet, pour éviter qu’une autre équipe commence à travailler sur la même règle que la vôtre.

Supposons que nous avons pris la règle suivante « Avoid creating getter and setter methods in classes », règle qui s’applique à du code Python.

Nous allons donc commencer à rechercher des exemples de code Python qui ne respectent pas cette règle, cela va nous permettre de coder la classe de test.

Pour vérifier si une règle que vous codez, n’est pas déjà présente, dans le catalogue des règles de sonarsource ou bien vous inspirer de règles déjà écrites pour en écrire une nouvelle, c’est [ici](https://rules.sonarsource.com/).

### Création du fichier de test
Dans la démarche TDD, créer un fichier de test va nous permettre d’identifier ce qu’il faudra contrôler dans les programmes en Python avec le plugin.

Il faut bien penser à mélanger du code qui respecte la règle et du code que ne la respecte pas, pour que les tests unitaires soient représentatifs.

Il faut aussi bien penser à indiquer sur chaque code qui ne respecte pas la règle les tags Noncompliant et {{Nom de votre règle}} qui sont utilisés par le plugin sonarqube.

La création du fichier de test et de la classe de règle sont les deux activités qui prennent le plus de temps à l’équipe qui développe une règle.

Le fichier de test nécessite de trouver des exemples de code qui respecte ou pas la règle et la classe de règle nécessite d’identifier des marqueurs dans l’AST du code pour déclencher dans le plugin l’affichage du non-respect de la règle.

Nous allons voir dans la suite de ce partage, comment fonctionne sonarqube et ce que représente l’AST d’un code.

Voilà ci-dessous à quoi ressemble le code de fichier de test, c’est bien entendu du code Python, puisque nous allons analyser des programmes écrits en Python.

```Python
from datetime import date
class Client():
    def __init__(self, age, weight):
        self.age = age
        self.weight = weight
    def set_age(self, age): # Noncompliant {{Avoid the use of getters and setters}}
        self.age = age
    def set_age(self, age2): # Noncompliant {{Avoid the use of getters and setters}}
        self.age = age2
    def get_age_in_five_years(self):
        a = Client()
        return a.age
    def get_age(self): # Noncompliant {{Avoid the use of getters and setters}}
        return self.age
    def is_major(self):
        return self.age >= 18
    def get_weight(self): # Noncompliant {{Avoid the use of getters and setters}}
        return self.weight
```

### Création de la classe de test
Maintenant que nous disposons de notre fichier de test, nous allons créer la classe de test qui va exécuter les tests unitaires qui sont dans le fichier de test.

Cette étape est très rapide, car cela sera toujours le même type de classe pour l’appel aux tests unitaires.

```Java
package fr.greencodeinitiative.python.checks;

import org.junit.Test;
import org.sonar.python.checks.utils.PythonCheckVerifier;

public class AvoidGettersAndSettersTest {

    @Test
    public void test() {
        PythonCheckVerifier.verify("src/test/resources/checks/avoidGettersAndSetters.py", new AvoidGettersAndSetters());
    }
}
```

### Création de la classe de règles
Sonarqube quand il analyse un programme crée un AST de ce programme.

Un arbre de syntaxe abstraite, ou AST, est une représentation arborescente du code source d'un programme informatique qui transmet la structure du code source et son contenu. Chaque nœud de l'arborescence représente une construction apparaissant dans le code source.

Tout le secret du développement d’une règle d’un plugin sonarqube et d’identifier dans l’AST du code analysé quel nœud il faut contrôler pour vérifier la véracité de la règle.

Pour vous aider vous pourrez utiliser cet outil [AST]( https://astexplorer.net/) et copier/coller le code d’exemple. C’est la partie qui prend le plus de temps, car il faut d’abord étudier l’arbre de syntaxe abstraite du code à analyser et identifier le nœud ou la configuration de nœud à tester pour savoir s’il faut déclencher la règle ou pas.

Voilà ce que cela donne pour la règle que nous avons choisie :
```Java
package fr.greencodeinitiative.python.checks;

import java.util.List;
import java.util.stream.Collectors;

import org.sonar.check.Priority;
import org.sonar.check.Rule;
import org.sonar.plugins.python.api.PythonSubscriptionCheck;
import org.sonar.plugins.python.api.SubscriptionContext;
import org.sonar.plugins.python.api.tree.AnyParameter;
import org.sonar.plugins.python.api.tree.AssignmentStatement;
import org.sonar.plugins.python.api.tree.FunctionDef;
import org.sonar.plugins.python.api.tree.ParameterList;
import org.sonar.plugins.python.api.tree.QualifiedExpression;
import org.sonar.plugins.python.api.tree.ReturnStatement;
import org.sonar.plugins.python.api.tree.Statement;
import org.sonar.plugins.python.api.tree.StatementList;
import org.sonar.plugins.python.api.tree.Tree;

@Rule(
        key = AvoidGettersAndSetters.RULE_KEY,
        name = "Developpement",
        description = AvoidGettersAndSetters.DESCRIPTION,
        priority = Priority.MINOR,
        tags = {"bug"})
public class AvoidGettersAndSetters extends PythonSubscriptionCheck {

    public static final String RULE_KEY = "D7";
    public static final String DESCRIPTION = "Avoid the use of getters and setters";

    @Override
    public void initialize(Context context) {
        context.registerSyntaxNodeConsumer(Tree.Kind.FUNCDEF, ctx -> {
            FunctionDef functionDef = (FunctionDef) ctx.syntaxNode();
            StatementList statementList = functionDef.body();
            List<Statement> statements = statementList.statements();
            if (functionDef.parent().parent().is(Tree.Kind.CLASSDEF)) {
                checkAllGetters(statements, functionDef, ctx);
                checkAllSetters(statements, functionDef, ctx);
            }
        });
    }

    public void checkAllSetters(List<Statement> statements, FunctionDef functionDef, SubscriptionContext ctx) {
        if (statements.size() == 1 && statements.get(0).is(Tree.Kind.ASSIGNMENT_STMT)) {
            AssignmentStatement assignmentStatement = (AssignmentStatement) statements.get(0);
            if (checkIfStatementIsQualifiedExpressionAndStartsWithSelfDot((QualifiedExpression) assignmentStatement.children().get(0).children().get(0))) {
                // Check if assignedValue is a parameter of the function
                ParameterList parameters = functionDef.parameters();
                if (parameters != null && !parameters.all().stream().filter(p -> checkAssignementFromParameter(assignmentStatement, p)).collect(Collectors.toList()).isEmpty()) {
                    ctx.addIssue(functionDef.defKeyword(), AvoidGettersAndSetters.DESCRIPTION);
                }
            }
        }
    }

    public void checkAllGetters(List<Statement> statements, FunctionDef functionDef, SubscriptionContext ctx) {
        Statement lastStatement = statements.get(statements.size() - 1);
        if (lastStatement.is(Tree.Kind.RETURN_STMT)) {
            List<Tree> returnStatementChildren = ((ReturnStatement) lastStatement).children();
            if (returnStatementChildren.get(1).is(Tree.Kind.QUALIFIED_EXPR) &&
                    checkIfStatementIsQualifiedExpressionAndStartsWithSelfDot((QualifiedExpression) returnStatementChildren.get(1))) {
                ctx.addIssue(functionDef.defKeyword(), AvoidGettersAndSetters.DESCRIPTION);
            }
        }
    }

    public boolean checkAssignementFromParameter(AssignmentStatement assignmentStatement, AnyParameter parameter) {
        String parameterToString = parameter.firstToken().value();
        return assignmentStatement.assignedValue().firstToken().value().equalsIgnoreCase(parameterToString);
    }

    public boolean checkIfStatementIsQualifiedExpressionAndStartsWithSelfDot(QualifiedExpression qualifiedExpression) {
        List<Tree> qualifedExpressionChildren = qualifiedExpression.children();
        return qualifedExpressionChildren.size() == 3 &&
                qualifedExpressionChildren.get(0).firstToken().value().equalsIgnoreCase("self") &&
                qualifedExpressionChildren.get(1).firstToken().value().equalsIgnoreCase(".");
    }
}
```

### Création du fichier de description de la règle
Cette étape sera aussi très rapide, car c’est toujours le même format de fichier, il faudra juste modifier les données liées à la description de la règle.

Il faudra indiquer le nom de la règle, le type de règle, sont statu et quelques informations de tag. Ces informations sont utilisées par sonarqube pour identifier la règle parmi toutes les règles disponibles dans l’outil et ses différents plugins.

Pour connaitre les recommandations de Sonar sur ce qu’il faut placer dans ce fichier, aller sur [ce lien]( https://docs.sonarcloud.io/digging-deeper/rules/).

```Json
{
    "title": "Avoid creating getter and setter methods in classes.",
    "type": "CODE_SMELL",
    "status": "ready",
    "remediation": {
      "func": "Constant\/Issue",
      "constantCost": "5min"
    },
    "tags": [
      "eco-conception"
    ],
    "defaultSeverity": "Minor"
  }
```
Avec cet exemple, nous avons un "titre" concis mais descriptif pour notre règle, le "type" de problème qu'il met en évidence, son "état" (prêt ou obsolète), les "balises" qui devraient le faire apparaître dans une recherche et le « gravité » du problème.

### Création du fichier de description des consignes aux développeurs
Nous devons d'abord remplir le fichier HTML avec des informations qui aideront les développeurs à résoudre le problème.

Ci-dessous nous avons une balise Noncompliant et un exemple de code correcte à écrire pour gérer correctement ses getters et setters.

Ces informations seront remontés au développeur, dans l'interface Sonar, si votre règle a identifiée une non conformité du code. 

```html
<p>Avoid Getters and Setters in class, It increase unless RAM memory usage.</p>
<h2>Noncompliant Code Example</h2>
<pre>
class Client():

    def __init__(self, age):
        self.age = age

    def get_age(self):
        return self.age

    def set_age(self, age):
        self.age = age

client = Client(25)
client.get_age() # Getter inutile
client.set_age(25) # Setter inutile

</pre>
<h2>Compliant Solution</h2>
<pre>
class Client():

    def __init__(self, age):
        self.age = age

    def get_age(self):
        return self.age

    def set_age(self, age):
        self.age = age

client = Client(25)
client.age # Récupérer l'attribut age
client.age = 26 # Modifier l'attribut age
</pre>
```

# Comment tester votre règle
À ce stade, nous avons terminé la mise en œuvre d'une première règle personnalisée et l'avons enregistrée dans le plug-in personnalisé. La dernière étape restante est de le tester directement avec la plateforme SonarQube et d'essayer d'analyser un projet !

Le projet ecoCode propose [ce guide](https://github.com/green-code-initiative/ecoCode-common/blob/main/doc/INSTALL.md) pour tester dans une version locale de Sonarqube le plugin développé, en utilisant docker.
