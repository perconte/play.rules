# Play et Scala

Play propose d'utiliser au choix les langages Scala ou Java pour développer des applications.

Play-Scala est plus qu'un simple module, c'est une version complète du framework dédiée à ce langage. Cette version possède un site dédié qui présente ses avantages et ses fonctionnalités : [scala.playframework.org](http://scala.playframework.org/)

Le code de l'application vote4music est disponible dans sa version Scala [ici](https://github.com/loicdescotte/vote4music-scala)

## Le langage Scala

Scala est un langage pour la machine virtuelle Java (JVM) qui marie les caractéristiques des langages orientés objet et des langage fonctionnels.
C'est un langage très différent de Java, il demande donc un temps d'adaptation pour les développeurs Java. Cependant, nous allons voir à travers quelques exemples que c'est un langage très intéressant qui peut nous aider à améliorer sensiblement la lisibilité et l'expressivité du code, grâce à l'approche fonctionnelle.

### Quelques exemples de code

On peut facilement filtrer une collection avec la fonction filter :

~~~ java
var numbers = Array(0, 1, 2, 3, 4, 5, 6, 7, 8, 9)
// Récupération des nombres pairs
var evenNumbers = numbers.filter(x => x%2==0)
~~~

On peut même simplifier l'écriture de cette fonction avec le caractère joker '_' :

~~~ java
var evenNumbers = numbers.filter(_%2==0)
~~~

Cette fonction nous sera utile dans l'application vote4music, notamment pour trier les albums par année :

~~~ java
albums = albums.filter(x => formatYear.format(x.releaseDate).equals(year))
~~~

Pour tirer des albums en fonction du nombre de votes, dans le sens décroissant, on peut écrire :

~~~ java
albums.sortBy(_.nbVotes).reverse
~~~

Pour trouver l'intervalle compris entre 2 entiers, ici le plus ancien album et le plus récent, il suffit d'écrire :

~~~ java
val first = Albums.firstAlbumYear
val last = Albums.lastAlbumYear
//Utilisation de la fonction to
years = first.to(last).toList
~~~

En Java, ces exemples auraient nécessité de passer par d'horribles boucles for (ou par l'utilisation de librairies comme Guava ou lambdaJ).

Voyons maintenant quelles sont les spécifités de Play-Scala et comment porter entièrement vote4music avec ce nouveau langage.

## Accès à la base de données avec l'API Anorm

Play-Scala intègre une API qui permet d'effectuer très facilement des requêtes SQL et de mapper les résultats dans des objets Scala :

### La classe Magic

En créant un objet Scala qui hérite de la classe `Magic`, on obtient des méthodes pour manipuler les objets dans la base.

Si on déclare :

~~~ java
object Album extends Magic[Album]
~~~

On peut par exemple écrire :

~~~ java
//récupération du premier élément
Album.find().first()
//recherche par genre
Album.find("genre = {g}").on("g" -> "ROCK").list()
//insertion
Album.create()
//mise à jour
Album.update()
~~~

### Requêtes plus complexes

On a souvent besoin de rammener plus d'un type d'objet à la fois. Cette méthode permet de récupérer tous les albums et les artistes dans la base de données :

~~~ java
def findAll:List[(Album,Artist)] =
      SQL(
       """
           select * from Album al
           join Artist ar on al.artist_id = ar.id
           order by al.nbVotes desc
           limit 100;
       """
      ).as( Album ~< Artist ^^ flatten * )
~~~

La dernière ligne utilise des 'matchers' définis par le framework pour mapper les résultats vers nos objets du modèle et les ranger dans une liste de pairs d'éléments (un album et son artiste).

Pour effectuer une recherche, on écrit la requête suivante :

~~~ java
def search(filter: String):List[(Album,Artist)] = {
      val likeFilter = "%".concat(filter).concat("%")
      SQL(
       """
           select * from Album al
           join Artist ar on al.artist_id = ar.id
           where al.name like {n}
           or ar.name like {n}
           order by al.nbVotes desc
           limit 100;
       """
      ).on("n"->likeFilter)
      .as( Album ~< Artist ^^ flatten * )
  }
~~~

La suppression des albums se déroule comme ceci :

def delete(id: Option[Long]) = {
    id.map(id => Album.delete("id={c}").onParams(id).executeUpdate())
    Action(Application.list)
  }

Option est un type Scala qui permet d'éviter les erreurs liées aux pointeurs nulls (NullPointerException). Quand on récupere un objet de type Option, il peut avoir la valeur `Some` ou `None`. La méthode `map` appliquée à cette option nous permet de traiter les résultats différents de `None`. 
La méthode `Action` permet d'effectuer une redirection vers une autre action du contrôleur.

## Le moteur de template

Play-Scala propose un moteur de template full Scala. Ce moteur permet d'écrire du code Scala autour de notre code HTML pour définir l'affichage des pages :

~~~ html
@(messages:List[String])

<ul>
@messages.map{ message =>
  <li>@message</li>
}
</ul>
~~~

La méthode map permet ici de parcourir la liste des éléments.

Pour effectuer le rendu d'un template, on appele une méthode qui porte le même nom que le le fichier HTML contenant ce template :

~~~ java
def index = html.index(messages)
~~~

Ces méthodes sont générées automatiquement par le compilateur.

	//TODO héritages de templates

### Quelques exemples concrets

Pour mettre à jour un album, on récupére l'album et l'artiste depuis la base de données puis on les transmet au template approprié :

~~~ java
def form(id: Option[Long]) = {
    val album = id.flatMap( id => Album.find("id={id}").onParams(id).first())
    val artist = album.flatMap( album => Artist.find("id={id}").onParams(album.artist_id).first())
    html.edit(album, artist)
}
~~~

On utilise ici flatMap pour récupérer un album ou un artiste à partir de son id. Si on avait utilisé map, on aurait récupéré une option contenant le résultat de la fonction passée en paramètre. flatMap permet de récupérer directement la valeur retournée (dans les cas différents de None) au lieu d'une option.

Pour parcourir une liste de résultats, par exemple un objet de type List[Album,Artist], on procède comme ceci :

~~~ java
  def list() = {
    html.list(Album.findAll)
  }
~~~

~~~ html
@(entries:List[(models.Album,models.Artist)])
@import controllers._
<table id="albumList">
    <thead>
        <tr>
            <th>Album</th>
            <th>Artist</th>
            <th>Cover</th>
            <th>Release date</th>
            <th>Genre</th>
            <th>Number of votes</th>
        </tr>
    </thead>

    @entries.map { entry =>
        <tr id="album-@entry._1.id">
            <td>@entry._1.name</td>
            <td>@entry._2.name</td>
            <td>
                @if(entry._1.hasCover){
                    <span class="cover"><a href="#">Show cover</a></span>
                }
            </td>
            <td>@Option(entry._1).map(_.releaseDate.format("yyyy-MM-dd"))</td>
            <td>@entry._1.genre</td>
            <td>
                <span id="nbVotes@entry._1.id">@entry._1.nbVotes</span>
            </td>
        </tr>
    }
</table>
~~~

Le template définie un paramètre `entries` qui correspond à la liste des tuples d'albums et d'artistes renvoyée par `Album.findAll`.
A l'interieur d'un tuple, on accède à un album avec l'expression `entry._1` et à un artiste avec `entry._2`.

	//TODO les tags

## Authentification à l'aide des traits

    // TODO

## Les tests

Le framework de test de Play-Scala est un bon d'exemple des avantages de ce langage. La syntaxe offerte par Scala donne des tests vraiment expressifs et simples à lire :

~~~ java	
test("collections") {
    var albums=Albums.findAll()
    (albums.size) should be (2)

    var artists=Artists.findAll()
    (artists.size) should be (1)

    Artist artist = artists(1)
    artist.name should include ("Joe")
}
~~~

Il est également possible d'écrire des tests dans le style BDD (Behavior Driven Developpment), en combinant le code des tests avec du texte représentant les comportements attendus :

~~~ java
val name = "Play.Rules"

"'Play.Rules'" should "not contain the X letter" in {
    name should not include ("X")
}

it should "have 10 chars" in {
    name should have length (10)
}
~~~

Si on applique ça à notre application, on peut par exemple écrire ce test : 

~~~ java
 it should "return right albums with years" in {

      val artist = new Artist("joe")

      val c1 = Calendar.getInstance()
      c1.set(2010,1,1)
      val album = new Album("album1", c1.getTime, "ROCK", false)

      val c2 = Calendar.getInstance()
      c2.set(2009,1,1)
      val album2 = new Album("album2", c2.getTime, "ROCK", false)

      val albums = List(new Tuple2(album,artist),new Tuple2(album2,artist))

      val filteredAlbums = Album.filterByYear(albums, "2010")
      filteredAlbums.size should be (1)
    }
~~~

NB : un Tuple2 est une paire d'éléments en Scala


## Installer le module Scala

Pour installer ce module, il suffit d'ajouter cette ligne dans le fichier `dependencies.yml`, dans la partie `require` : `- play -> scala 0.9.1`.
Ca y'est vous êtes armés pour développer en Scala!

## Apprendre Scala

Si vous désirez apprendre ce langage, il existe [un e-book gratuit](http://programming-scala.labs.oreilly.com/index.html) (en anglais)
	
[Home page du module Scala](http://scala.playframework.org/)