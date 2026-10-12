# Tuto Java : comprendre les Streams

> Tutoriel pour débutants, basé sur la vidéo **« Better Java Streams with Gatherers – JEP Cafe #23 »**, limité à la partie avant **42:30**.  
> Il couvre les Streams, les opérations intermédiaires et terminales, les Collectors et les Streams parallèles.

> **Version Java recommandée : Java 21 ou plus récent.**  
> Les exemples utilisent notamment `record` et `Stream.toList()`, disponibles depuis Java 16.

---

## Sommaire

1. [C’est quoi un Stream ?](#1-cest-quoi-un-stream)
2. [Une seule utilisation par Stream](#2-une-seule-utilisation-par-stream)
3. [Opérations intermédiaires et terminales](#3-opérations-intermédiaires-et-terminales)
4. [`map()` : transformer chaque élément](#4-map--transformer-chaque-élément)
5. [`filter()` : garder certains éléments](#5-filter--garder-certains-éléments)
6. [`flatMap()` : aplatir des données](#6-flatmap--aplatir-des-données)
7. [`mapMulti()` : alternative plus efficace à `flatMap()`](#7-mapmulti--alternative-plus-efficace-à-flatmap)
8. [`limit()` et `takeWhile()` : arrêter un Stream](#8-limit-et-takewhile--arrêter-un-stream)
9. [`skip()` et `dropWhile()` : retarder le traitement](#9-skip-et-dropwhile--retarder-le-traitement)
10. [État interne : pourquoi c’est important](#10-état-interne-pourquoi-cest-important)
11. [`sorted()` et `distinct()` : attention à la mémoire](#11-sorted-et-distinct--attention-à-la-mémoire)
12. [`peek()` : uniquement pour déboguer](#12-peek--uniquement-pour-déboguer)
13. [Les opérations terminales utiles](#13-les-opérations-terminales-utiles)
14. [Les opérations “court-circuit”](#14-les-opérations-court-circuit)
15. [`collect()` et les Collectors](#15-collect-et-les-collectors)
16. [Créer ton propre Collector](#16-créer-ton-propre-collector)
17. [Les Streams parallèles](#17-les-streams-parallèles)
18. [Mini-exercice](#18-mini-exercice)
19. [À retenir](#19-à-retenir)

---

## 1. C’est quoi un Stream ?

Un `Stream` est une **chaîne de traitement de données** : tu lui donnes une source (liste, tableau, fichier, etc.), puis tu appliques des transformations, et enfin tu récupères un résultat.

> Important : un Stream **ne stocke pas les données**. Il les récupère depuis une source, comme une `List`, un tableau, les lignes d’un fichier, ou même un générateur de nombres.

### Exemple simple

```java
import java.util.List;

public class Main {
    public static void main(String[] args) {
        List<String> noms = List.of("alice", "bob", "carole");

        List<String> resultats = noms.stream()
                .map(String::toUpperCase)
                .toList();

        System.out.println(resultats);
        // [ALICE, BOB, CAROLE]
    }
}
```

### Ce qui se passe ici

- `noms.stream()` crée un Stream à partir de la liste.
- `map(...)` transforme chaque élément.
- `toList()` récupère le résultat dans une nouvelle liste.

---

## 2. Une seule utilisation par Stream

Un Stream ne peut être utilisé **qu’une seule fois**. Si tu essaies d’appeler une deuxième opération sur le même objet `Stream`, Java lève une exception :

```text
IllegalStateException: stream has already been operated upon or closed
```

### Bonne pratique

Écris plutôt les opérations **enchaînées** :

```java
List<String> resultats = noms.stream()
        .map(String::toUpperCase)
        .toList();
```

### À éviter

```java
var stream = noms.stream();
stream.map(String::toUpperCase); // nouveau Stream
stream.toList(); // ❌ erreur : le Stream initial a déjà été utilisé
```

> Chaque opération intermédiaire crée un **nouveau Stream** ; elle ne modifie jamais le Stream d’origine.

---

## 3. Opérations intermédiaires et terminales

Il y a deux grandes familles de méthodes.

| Type | Retourne | Exemples | Rôle |
|---|---|---|---|
| Intermédiaire | Un nouveau `Stream` | `map()`, `filter()`, `flatMap()` | Prépare les transformations |
| Terminale | Autre chose : liste, nombre, `Optional`, `void`… | `toList()`, `count()`, `forEach()` | Déclenche le calcul |

- Les méthodes **intermédiaires** renvoient toujours un Stream.
- Les méthodes **terminales** renvoient un résultat final, une valeur, ou rien du tout.

### Le piège principal : rien ne s’exécute sans opération terminale

Un Stream est **paresseux** (*lazy*) : il ne traite les données que lorsqu’on appelle une opération terminale.

```java
noms.stream()
    .map(nom -> {
        System.out.println("Traitement de : " + nom);
        return nom.toUpperCase();
    });
```

Ce code n’affiche **rien** : il n’y a pas d’opération terminale. Le Stream existe, mais ne fait rien.

Ajoute `toList()` :

```java
List<String> resultats = noms.stream()
    .map(nom -> {
        System.out.println("Traitement de : " + nom);
        return nom.toUpperCase();
    })
    .toList();
```

Là, les messages apparaissent, car `toList()` déclenche le traitement.

---

## 4. `map()` : transformer chaque élément

`map()` prend un élément et renvoie généralement un autre élément. C’est une transformation **1 vers 1**.

```java
List<String> prenoms = List.of("alice", "bob");

List<Integer> longueurs = prenoms.stream()
        .map(String::length)
        .toList();

System.out.println(longueurs); //[3][5]
```

Tu peux aussi utiliser une lambda classique :

```java
List<String> majuscules = prenoms.stream()
        .map(prenom -> prenom.toUpperCase())
        .toList();
```

---

## 5. `filter()` : garder certains éléments

`filter()` garde seulement les éléments pour lesquels la condition est vraie.

```java
List<Integer> nombres = List.of(1, 2, 3, 4, 5, 6);

List<Integer> pairs = nombres.stream()
        .filter(n -> n % 2 == 0)
        .toList();

System.out.println(pairs); //[2][4][6]
```

> `filter()` ne modifie pas les éléments : il décide seulement s’ils passent à la suite ou non.

---

## 6. `flatMap()` : aplatir des données

`flatMap()` est utile quand chaque élément contient **plusieurs éléments**.  
Par exemple : une liste de pays, où chaque pays possède plusieurs villes.

```java
import java.util.List;

record Pays(String nom, List<String> villes) {}

public class Main {
    public static void main(String[] args) {
        List<Pays> pays = List.of(
            new Pays("Belgique", List.of("Bruxelles", "Charleroi")),
            new Pays("France", List.of("Paris", "Lille"))
        );

        List<String> toutesLesVilles = pays.stream()
                .flatMap(p -> p.villes().stream())
                .toList();

        System.out.println(toutesLesVilles);
        // [Bruxelles, Charleroi, Paris, Lille]
    }
}
```

> Sans `flatMap()`, tu aurais une liste de listes. Avec, tu obtiens une seule liste de villes.

---

## 7. `mapMulti()` : alternative plus efficace à `flatMap()`

`flatMap()` crée souvent un Stream intermédiaire pour chaque élément.  
`mapMulti()` évite cela : tu envoies directement les éléments vers la suite du Stream avec `downstream.accept(...)`.

```java
List<String> toutesLesVilles = pays.stream()
        .<String>mapMulti((p, downstream) -> {
            for (String ville : p.villes()) {
                downstream.accept(ville);
            }
        })
        .toList();
```

### Comment ça fonctionne

`mapMulti()` prend deux paramètres :

- l’élément courant du Stream ;
- le `downstream`, c’est-à-dire la suite du traitement.

Tu peux appeler `downstream.accept()` **zéro, une ou plusieurs fois**. C’est donc aussi utile pour valider des données : si une ligne est invalide, tu n’appelles pas `accept()`.

### Exemple : convertir des lignes en nombres

```java
import java.util.List;

public class Main {
    public static void main(String[] args) {
        List<String> lignes = List.of("10", "abc", "20", "# commentaire", "30");

        List<Integer> nombres = lignes.stream()
                .<Integer>mapMulti((ligne, downstream) -> {
                    try {
                        downstream.accept(Integer.parseInt(ligne));
                    } catch (NumberFormatException e) {
                        // on ignore la ligne invalide
                    }
                })
                .toList();

        System.out.println(nombres); //[10][20][30]
    }
}
```

> Ici, la conversion n’est faite qu’une seule fois, contrairement à une solution avec `filter()` puis `map()`.

---

## 8. `limit()` et `takeWhile()` : arrêter un Stream

Certaines opérations intermédiaires peuvent **interrompre** le Stream : elles arrêtent de consommer la source, même s’il reste des données.

### `limit(n)`

Garde seulement les `n` premiers éléments.

```java
List<Integer> nombres = List.of(1, 2, 3, 4, 5, 6);

List<Integer> troisPremiers = nombres.stream()
        .limit(3)
        .toList();

System.out.println(troisPremiers); //[1][2][3]
```

### `takeWhile(condition)`

Garde les éléments tant que la condition est vraie. Dès qu’elle devient fausse, le Stream s’arrête.

```java
List<Integer> nombres = List.of(1, 2, 3, 10, 4, 5);

List<Integer> resultat = nombres.stream()
        .takeWhile(n -> n < 10)
        .toList();

System.out.println(resultat); //[1][2][3]
```

> Même si `4` et `5` sont inférieurs à 10, ils ne sont pas pris : le Stream s’est arrêté dès `10`.

---

## 9. `skip()` et `dropWhile()` : retarder le traitement

Ces méthodes font l’inverse : elles ignorent le début du Stream.

### `skip(n)`

Ignore les `n` premiers éléments.

```java
List<Integer> nombres = List.of(1, 2, 3, 4, 5);

List<Integer> resultat = nombres.stream()
        .skip(2)
        .toList();

System.out.println(resultat); //[3][4][5]
```

### `dropWhile(condition)`

Ignore les éléments tant que la condition est vraie. Une fois qu’elle devient fausse, il garde tout le reste.

```java
List<Integer> nombres = List.of(1, 2, 3, 10, 4, 5);

List<Integer> resultat = nombres.stream()
        .dropWhile(n -> n < 10)
        .toList();

System.out.println(resultat); //[4][5][10]
```

> Après la première valeur qui casse la condition, `dropWhile()` ne re-teste plus la condition.

---

## 10. État interne : pourquoi c’est important

`limit()`, `skip()`, `takeWhile()` et `dropWhile()` doivent garder une information en mémoire :

- `limit()` et `skip()` comptent les éléments déjà vus.
- `takeWhile()` et `dropWhile()` se souviennent s’ils sont encore “ouverts” ou “fermés”.

Cela s’appelle un **état interne mutable**.

> C’est un concept important pour comprendre plus tard les *Gatherers*, mais pour l’instant, retiens juste que certaines opérations ont besoin de mémoriser des informations pendant le traitement.

---

## 11. `sorted()` et `distinct()` : attention à la mémoire

### `sorted()`

`sorted()` doit voir **tous les éléments** avant de pouvoir en envoyer un seul, car il doit d’abord les trier.

```java
List<Integer> nombres = List.of(5, 1, 4, 2);

List<Integer> tries = nombres.stream()
        .sorted()
        .toList();

System.out.println(tries); //[1][2][4][5]
```

> Ne l’utilise pas sur un Stream infini : il essaierait de stocker une infinité d’éléments en mémoire.

### `distinct()`

`distinct()` supprime les doublons.

```java
List<String> mots = List.of("java", "stream", "java", "code");

List<String> sansDoublons = mots.stream()
        .distinct()
        .toList();

System.out.println(sansDoublons); // [java, stream, code]
```

> Il utilise une structure interne de type `Set` pour savoir si un élément a déjà été vu.

---

## 12. `peek()` : uniquement pour déboguer

`peek()` permet de voir les éléments qui passent dans le Stream, sans les modifier.

```java
List<String> resultats = noms.stream()
        .peek(nom -> System.out.println("Avant map : " + nom))
        .map(String::toUpperCase)
        .toList();
```

### Attention

`peek()` est prévu pour le débogage, pas pour modifier des données extérieures.

Évite ceci :

```java
List<String> vus = new ArrayList<>();

noms.stream()
    .peek(vus::add) // ❌ mauvaise idée
    .toList();
```

> La raison : Java peut optimiser le Stream et ne pas appeler toutes les lambdas. Si tu ajoutes des effets de bord dans `peek()`, ton code peut avoir un comportement imprévisible.

---

## 13. Les opérations terminales utiles

### `forEach()`

Exécute une action pour chaque élément.

```java
noms.stream()
    .forEach(nom -> System.out.println(nom));
```

> C’est pratique pour afficher ou déboguer, mais ce n’est généralement pas la meilleure façon de construire un résultat. Pour récupérer une liste, préfère `toList()`.

### `count()`, `min()`, `max()`

```java
long nombre = noms.stream().count();

Optional<String> plusCourt = noms.stream()
        .min(Comparator.comparingInt(String::length));

Optional<String> plusLong = noms.stream()
        .max(Comparator.comparingInt(String::length));
```

> `min()`, `max()` et `average()` renvoient un `Optional`, car le Stream peut être vide : il n’y a alors ni minimum, ni maximum, ni moyenne.

### `sum()` et `average()`

Ces méthodes existent sur les Streams spécialisés : `IntStream`, `LongStream` et `DoubleStream`.

```java
import java.util.List;

public class Main {
    public static void main(String[] args) {
        List<Integer> nombres = List.of(1, 2, 3, 4);

        int somme = nombres.stream()
                .mapToInt(Integer::intValue)
                .sum();

        System.out.println(somme); // 10
    }
}
```

---

## 14. Les opérations “court-circuit”

Certaines opérations terminales s’arrêtent dès qu’elles ont leur réponse. Elles ne consomment donc pas forcément tout le Stream.

| Méthode | S’arrête quand… |
|---|---|
| `findFirst()` | Elle trouve le premier élément |
| `findAny()` | Elle trouve un élément |
| `allMatch()` | Elle trouve un élément qui ne respecte pas la condition |
| `anyMatch()` | Elle trouve un élément qui respecte la condition |
| `noneMatch()` | Elle trouve un élément qui respecte la condition |

### Exemples

```java
boolean tousPositifs = nombres.stream()
        .allMatch(n -> n > 0);

boolean auMoinsUnPair = nombres.stream()
        .anyMatch(n -> n % 2 == 0);

Optional<Integer> premier = nombres.stream()
        .findFirst();
```

> Ces méthodes sont appelées *short-circuiting* : elles peuvent renvoyer un résultat sans parcourir toute la source.

---

## 15. `collect()` et les Collectors

`collect()` est une opération terminale qui permet de rassembler les éléments du Stream dans un conteneur : liste, ensemble, chaîne de caractères, map, etc.

> Attention : `Collectors` contient les Collectors déjà prêts, comme `toList()`, `toSet()`, `joining()` et `groupingBy()`.  
> En revanche, pour créer un Collector personnalisé, on utilise `Collector.of(...)`, sur l’interface `Collector`.

### `toList()`

```java
List<String> resultats = noms.stream()
        .map(String::toUpperCase)
        .toList();
```

### `toSet()`

```java
Set<String> uniques = noms.stream()
        .collect(Collectors.toSet());
```

### `joining()`

Pour les Streams de `String` :

```java
String phrase = List.of("Java", "Streams", "sont", "utiles").stream()
        .collect(Collectors.joining(" "));

System.out.println(phrase); // Java Streams sont utiles
```

Tu peux aussi ajouter un séparateur, un préfixe et un suffixe :

```java
String phrase = List.of("a", "b", "c").stream()
        .collect(Collectors.joining(", ", "[", "]"));

System.out.println(phrase); // [a, b, c]
```

### `groupingBy()`

C’est l’équivalent d’un `GROUP BY` en SQL : il regroupe les éléments selon une clé.

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

record Personne(String nom, String ville) {}

public class Main {
    public static void main(String[] args) {
        List<Personne> personnes = List.of(
            new Personne("Alice", "Charleroi"),
            new Personne("Bob", "Bruxelles"),
            new Personne("Carole", "Charleroi")
        );

        Map<String, List<Personne>> parVille = personnes.stream()
                .collect(Collectors.groupingBy(Personne::ville));

        System.out.println(parVille);
    }
}
```

Résultat approximatif :

```text
{
  Charleroi=[Personne[nom=Alice, ville=Charleroi], Personne[nom=Carole, ville=Charleroi]],
  Bruxelles=[Personne[nom=Bob, ville=Bruxelles]]
}
```

> `groupingBy()` peut aussi être combiné avec un autre Collector, par exemple pour compter les éléments de chaque groupe.

---

## 16. Créer ton propre Collector

Un Collector repose généralement sur :

1. Un `Supplier` : crée le conteneur.
2. Un `accumulator` : ajoute un élément au conteneur.
3. Un `combiner` : fusionne deux conteneurs, surtout pour le parallélisme.
4. Un `finisher` : transforme le conteneur en résultat final, si nécessaire.

### Exemple minimal : collecter dans une `ArrayList`

```java
import java.util.List;
import java.util.ArrayList;
import java.util.stream.Collector;

public class Main {
    public static void main(String[] args) {
        List<String> noms = List.of("alice", "bob");

        List<String> resultats = noms.stream()
                .collect(Collector.of(
                        ArrayList::new,
                        ArrayList::add,
                        (gauche, droite) -> {
                            gauche.addAll(droite);
                            return gauche;
                        }
                ));

        System.out.println(resultats); // [alice, bob]
    }
}
```

> Ici, `ArrayList::new` crée le conteneur, `ArrayList::add` ajoute chaque élément, et la lambda finale fusionne deux conteneurs partiels. C’est le `combiner`, utile notamment pour les Streams parallèles.

### Le `finisher`

Certaines collections ne sont pas modifiables, comme `String`. Pour créer une `String`, un Collector peut d’abord remplir un `StringBuilder`, puis utiliser un `finisher` pour appeler `toString()`.

> Le `finisher` sert aussi à produire une liste ou une map **non modifiable** à la fin.

---

## 17. Les Streams parallèles

Un Stream peut être parallélisé avec `.parallelStream()` ou `.parallel()`.

```java
List<Integer> nombres = List.of(1, 2, 3, 4, 5, 6);

long total = nombres.parallelStream()
        .filter(n -> n % 2 == 0)
        .count();

System.out.println(total); // 3
```

> Java découpe alors les données en petits morceaux, les traite dans plusieurs threads, puis fusionne les résultats partiels.

### Quand le parallélisme fonctionne bien

Il fonctionne mieux quand :

- la source peut être découpée en deux parties équilibrées ;
- la source est `SIZED` : Java connaît à l’avance le nombre d’éléments ;
- la source est `SUBSIZED` : après découpage, Java connaît aussi la taille de chaque partie.

> Une `ArrayList` est généralement à la fois `SIZED` et `SUBSIZED`, donc adaptée au parallélisme. Une source comme les lignes d’un fichier ne l’est généralement pas.

### Les dangers du parallélisme

Le parallélisme n’est pas automatiquement plus rapide. Il peut même être plus lent ou dangereux si :

- tes lambdas modifient une variable extérieure ;
- tu ajoutes des éléments dans une `ArrayList` non thread-safe depuis plusieurs threads ;
- ton opération de réduction n’est pas associative ;
- ton application utilise déjà beaucoup le CPU.

Par exemple, ceci est dangereux :

```java
List<Integer> resultats = new ArrayList<>();

nombres.parallelStream()
       .filter(n -> n % 2 == 0)
       .forEach(resultats::add); // ❌ risque de race condition
```

Plusieurs threads peuvent ajouter des éléments dans la même `ArrayList` en même temps.

Préfère :

```java
List<Integer> resultats = nombres.parallelStream()
        .filter(n -> n % 2 == 0)
        .toList();
```

> Une opération de réduction doit être **associative** : peu importe comment les calculs partiels sont regroupés, le résultat final doit rester correct.

---

## 18. Mini-exercice

Écris un programme qui :

1. Prend une liste de phrases.
2. Les découpe en mots.
3. Garde les mots de plus de 3 lettres.
4. Les met en majuscules.
5. Les regroupe par première lettre.
6. Affiche le résultat.

### Correction possible

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class Main {
    public static void main(String[] args) {
        List<String> phrases = List.of(
            "Java est puissant",
            "Les streams simplifient le code"
        );

        Map<Character, List<String>> resultat = phrases.stream()
                .flatMap(phrase -> List.of(phrase.split(" ")).stream())
                .filter(mot -> mot.length() > 3)
                .map(String::toUpperCase)
                .collect(Collectors.groupingBy(mot -> mot.charAt(0)));

        System.out.println(resultat);
    }
}
```

---

## 19. À retenir

- Un Stream est une chaîne de traitement, pas un conteneur de données.
- Les opérations intermédiaires préparent le traitement ; l’opération terminale le déclenche.
- `map()` transforme, `filter()` sélectionne, `flatMap()` et `mapMulti()` aplatissent.
- `limit()` et `takeWhile()` peuvent arrêter un Stream.
- `collect()` permet de construire le résultat final avec un `Collector`.
- Les Streams parallèles peuvent accélérer certains traitements, mais seulement si la source, les opérations et l’environnement s’y prêtent.

> La suite de la vidéo, après 42:30, introduit les **Gatherers** : une API pour créer tes propres opérations intermédiaires de Stream, notamment pour faire des fenêtres glissantes ou des traitements avec état.
