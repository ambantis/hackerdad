+++
title = "Using a Database View with Scala Slick"
date = 2013-10-01
path = "blog/2013/10/01/using-a-database-view-with-scala-slick"
+++

Scala Slick is pretty darn cool. I’ve been using it for my app MagicNotebook.io, which is going to change the way teachers guide students through the writing process.

As I explored in my last post, it is important that design decisions should drive technology choices and not the other way around. To do that, you need to be prepared to think outside the box regarding what exactly those technology choices are, and that’s the focus of today’s post.

To review, here’s the current representation of a Google Drive folder in my app:

```scala
case class GFolder(id: String,
                   eTag: String,
                   url: String,
                   iconUrl: String,
                   title: String,
                   owner: String,
                   parents: Set[String],
                   children: Set[String],
                   scions: Set[String],
                   created: LocalDateTime,
                   modified: LocalDateTime) extends GFile {
  def folders(implicit gFolders: Map[String, GFolder]): List[GFolder] = {...}

  def docs(implicit gDocs: Map[String, GDoc]): List[GDoc] = {...}

  def export(implicit gFolderMap: Map[String, GFolder],
                      gDocMap: Map[String, GDoc]): List[TreeTableRow] = {...}
```

And here’s the object that is persisted to cache so the user can work with his/her Google Drive files:

```scala
case class MagicNotebook(tree: GFolder,
                         implicit val folders: Map[String, GFolder],
                         implicit val docs: Map[String, GDoc])

object MagicNotebook {

  def create(userId: String)(implicit s: Session) = {
    DbHomes.findById(userId).flatMap { fileId =>
      DbGFolders.findById(fileId).map { tree =>
        val folders = DbGFolders.findAllFor(fileId)
        val docs = DbGDocs.findAllFor(fileId)
        MagicNotebook(tree, folders, docs)
      }
    }
  }
  val empty = MagicNotebook(GFolder.empty, Map(), Map())
}
```

To create a MagicNotebook object, we start with the user’s id to get the fileId of the Google Drive folder that is the root of all the MagicNotebook files in the user’s Google Drive. With that, we build a GFolder object of that root folder along with a map of all the subdirectory GFolders and GDocs. But in the database, the representation of a GFolder is split among four tables. In addition, there is scions, representing the children, grand-children, great-grand-children, etc of a folder.

My first attempt to do this transformation was purely with Scala and Scala Slick code, and seemed a bit clunky. So it was time to think outside the box. The documentation for Scala Slick explains clearly how to map a Scala object to a database table, but nowhere in the documentation does it mention database views. I tested it with Play 2.2.0 and Scala Slick 1.0.1, and it turns out you can bind a Scala Slick Table to a database view! First, we begin by defining a view that uses a recursive query to find all scions (children, grand-children, great-grand-children, etc.) of a folder:

```sql
CREATE VIEW scion_view AS
  WITH RECURSIVE scions(id, scion) AS (
    SELECT c.id, c.child
    FROM children AS c
    UNION ALL
    SELECT s.id, c.child
    FROM children AS c, scions AS s
    WHERE c.id = s.scion)
  SELECT * FROM scions ORDER BY id, scion;
```

Next, we need to [string-agg](http://www.postgresql.org/docs/9.2/static/functions-aggregate.html) function that will flatten multiple rows of a fileIds from a one-to-many relationship to a one-to-one relationship as a single comma-delimited string row:

```sql
SELECT DISTINCT
  id, string_agg(child, ',' ORDER BY child) AS child_str
FROM children GROUP BY id;
```

With these two building blocks in place, we can create the view that maps directly to the Scala GFolder object:

```sql
CREATE VIEW gfolder_view AS
  SELECT
    f.id, f.e_tag, f.url, f.icon_url, f.title, m.name, f.file_owner,
    p.parent_str, c.child_str, s.scion_str, f.created, f.modified
  FROM
    gfiles AS f
      JOIN mimes AS m ON (f.mime_type = m.name)
      LEFT JOIN (SELECT DISTINCT id, string_agg(parent, ',' ORDER BY parent) AS parent_str
                 FROM parents GROUP BY id) AS p ON (f.id = p.id)
      LEFT JOIN (SELECT DISTINCT id, string_agg(child, ',' ORDER BY child) AS child_str
                 FROM children GROUP BY id) AS c ON (f.id = c.id)
      LEFT JOIN (SELECT DISTINCT id, string_agg(scion, ',' ORDER BY scion) AS scion_str
                 FROM scion_view GROUP BY id) AS s ON (f.id = s.id)
  WHERE
    m.category = 'folder';
```

The only transformation required to map this view into a GFolder object is to split the comma-delimited String of parent, child, & scion fileIds, which is rather trivial to do with Scala. So here is the Scala object:

```scala
case class DbGFolder(id: String,
                     eTag: String,
                     url: String,
                     iconUrl: String,
                     title: String,
                     owner: String,
                     parents: Option[String],
                     children: Option[String],
                     scions: Option[String],
                     created: LocalDateTime,
                     modified: LocalDateTime)

object DbGFolders extends Table[DbGFolder]("gfolder_view") {
  def id = column[String]("id")
  def eTag = column[String]("e_tag")
  def url = column[String]("url")
  def iconUrl = column[String]("icon_url")
  def title = column[String]("title")
  def owner = column[String]("file_owner")
  def parents = column[String]("parent_str")
  def children = column[String]("child_str")
  def scions = column[String]("scion_str")
  def created = column[LocalDateTime]("created")
  def modified = column[LocalDateTime]("modified")
  def * = id ~ eTag ~ url ~ iconUrl ~ title ~ owner ~ parents.? ~
          children.? ~ scions.? ~ created ~ modified
          <> (DbGFolder, DbGFolder.unapply _)

  def findAllFor(root: String)(implicit s: Session): Map[String, GFolder] = {
    val query = for {
      (_, gFolder) <- DbScions.innerJoin(DbGFolders)
                              .on(_.scion === _.id)
                              .where(_._1.id === root)
    } yield gFolder

    query.list().map {v =>
      GFolder(v.id,
              v.eTag,
              v.url,
              v.iconUrl,
              v.title,
              v.owner,
              v.parents.map { parentStr =>
                parentStr.split(",").toSet }.getOrElse(Set()),
              v.children.map{ childStr =>
                childStr.split(",").toSet }.getOrElse(Set()),
              v.scions.map { scionStr =>
                scionStr.split(",").toSet }.getOrElse(Set()),
              v.created,
              v.modified)
    }.groupBy(_.id).mapValues(_.head)
  }

  def findById(fileId: String)(implicit s: Session): Option[GFolder] = {
    Query(DbGFolders).filter(_.id === fileId).list().headOption.map {v =>
      GFolder(v.id,
              v.eTag,
              v.url,
              v.iconUrl,
              v.title,
              v.owner,
              v.parents.map { parentStr =>
                parentStr.split(",").toSet }.getOrElse(Set()),
              v.children.map{ childStr =>
                childStr.split(",").toSet }.getOrElse(Set()),
              v.scions.map { scionStr =>
                scionStr.split(",").toSet }.getOrElse(Set()),
              v.created,
              v.modified)
    }
  }
}
```

Thus, moving the application logic to denormalize a GFolder back to postgres allowed me to significantly improve the Scala code readability.
