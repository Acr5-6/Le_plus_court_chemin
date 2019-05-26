# Le_plus_court_chemin
#Projet 4A : Fondement et application des bases de données graphes


#Télécharger ce dossier.


#Télécharger les données OSM : https://download.bbbike.org/osm/bbbike/Marseille/Marseille.osm.gz


#Décompressez le fichier et placez-le dans le dossier osm_import_Neo4j/samples, préalablement créé.


#Placez-vous dans le dossier osm_import_Neo4j.


#Comme c’est un projet Maven, il faut compiler l’application Java grâce à la commande : 
```
mvn clean install
```

#pour compiler sans test:
```
mvn clean install -DskipTests
```

#Ensuite nous devons obtenir toutes les dépendances nécessaires pour l’application : 
mvn dependency:copy-dependencies


#Décompressez le fichier plugins.zip dans le dossier neo4j-community-X.X.X\plugins de votre système.


#En fonction de votre système, effectuez l'une de ces deux commandes suivantes :
#Windows
```
java -Xms1280m -Xmx1280m -cp target\osm-0.2.2-neo4j-3.5.1.jar;target\dependency\* org.neo4j.gis.osm.OSMImportTool --skip-duplicate-nodes --delete --into target\databases\marseille samples\Marseille.osm
```

#Linux
```
java -Xms1280m -Xmx1280m -cp "target/osm-0.2.2-neo4j-3.5.1.jar:target/dependency/*" org.neo4j.gis.osm.OSMImportTool --skip-duplicate-nodes --delete --into target/databases/marseille samples/Marseille.osm.
```

#déplacez le dossier target/databases/marseille dans neo4j-community-X.X.X/data/databases

#Activer cette base de données dans neo4j-community-3.5.5/conf/neo4j.conf:
```
# The name of the database to mount
dbms.active_database=marseille
```

## Traitement
#Lancez neo4j:
```
neo4j console
```

#Il faut référencer le path de neo4j dans la variable d'environnement $PATH.


#Lancez ces requêtes sur la base de données avec Neo4j afin d'obtenir des graphes pouvant être utilisés par notre application.

```
MATCH (awn:OSMWayNode)-[r:NEXT]-(bwn:OSMWayNode)
WHERE NOT exists(r.distance)
WITH awn, bwn, r LIMIT 10000
MATCH (awn)-[:NODE]->(a:OSMNode), (bwn)-[:NODE]->(b:OSMNode)
SET r.distance= distance(a.location,b.location)
RETURN COUNT(*)
```
```
CALL apoc.periodic.iterate('MATCH (awn:OSMWayNode)-[r:NEXT]-(bwn:OSMWayNode) WHERE NOT exists(r.distance) RETURN awn, bwn, r',
'MATCH (awn)-[:NODE]->(a:OSMNode), (bwn)-[:NODE]->(b:OSMNode)
SET r.distance= distance(a.location,b.location)', {batchSize:10000,parallel:false});
```
```
MATCH (n:OSMNode)
WHERE size((n)<-[:NODE]-(:OSMWayNode)-[:NEXT]-(:OSMWayNode)) > 2
AND NOT (n:Intersection)
MATCH (n)<-[:NODE]-(wn:OSMWayNode), (wn)<-[:NEXT*0..100]-(wx), (wx)<-[:FIRST_NODE]-(w:OSMWay)-[:TAGS]->(wt:OSMTags) WHERE exists(wt.highway)
SET n:Intersection
RETURN count(*);
```
```
MATCH (x:Intersection) 
CALL spatial.osm.routeIntersection(x,false,false,false) 
YIELD fromNode, toNode, fromRel, toRel, distance, length, count 
WITH fromNode, toNode, fromRel, toRel, distance, length, count
MERGE (fromNode)-[r:ROUTE {fromRel:id(fromRel),toRel:id(toRel)}]->(toNode)
ON CREATE SET r.distance = distance, r.length = length, r.count = count
RETURN count(*);
```
```
UNWIND ["restaurant","fast_food","cafe","bar","pub","ice_cream","cinema"] AS amenity
MATCH (x:OSMNode)-[:TAGS]->(t:OSMTags)
  WHERE t.amenity = amenity AND NOT (x)-[:ROUTE]->()
WITH x, x.location as poi LIMIT 100
MATCH (n:OSMNode)
  WHERE distance(poi, n.location) < 100
WITH x, n
MATCH (n)<-[:NODE]-(wn:OSMWayNode), (wn)<-[:NEXT*0..10]-(wx),
      (wx)<-[:FIRST_NODE]-(w:OSMWay)-[:TAGS]->(wt:OSMTags)
WITH x, w, wt
  WHERE exists(wt.highway)
WITH x, collect(w) as ways
  CALL spatial.osm.routePointOfInterest(x,ways) YIELD node
  SET x:PointOfInterest
RETURN count(node);
```
```
MATCH (x:Routable:OSMNode)
WHERE NOT (x)-[:ROUTE]->(:Intersection) WITH x LIMIT 100
CALL spatial.osm.routeIntersection(x,true,false,false)
YIELD fromNode, toNode, fromRel, toRel, distance, length, count
WITH fromNode, toNode, fromRel, toRel, distance, length, count
```
```
MATCH (a:PointOfInterest)
WITH a,a.name + toString(a.location.latitude) + toString(a.location.longitude) AS poi
SET a.poi_id = poi
```

## Requêtes utiles pour le design et la fonctionnalité A* de l'application

#Ajouter l'attribut "amenity" pour les POI ayant une relation TAGS à un amenity
```
MATCH (n:PointOfInterest)-[:TAGS]->(o:OSMTags) 
WHERE NOT EXISTS(n.amenity) 
SET n.amenity=o.amenity 
RETURN n;
```

#Icon pub n'existe pas -> pub = bar
```
MATCH (n:PointOfInterest)
WHERE (n.amenity=pub)
SET n.amenity=bar
RETURN n;
```

### ASTAR
#Ajouter les attributs "lat" et "lon" pour les noeuds ROOTABLE 
```
MATCH (n:Routable)<-[:ROUTE]-(o:PointOfInterest) 
WHERE NOT EXISTS(n.lat) AND NOT EXISTS(n.lon) 
SET n.lat=o.lat,n.lon=o.lon 
RETURN n;
```

## Application
#Initialisez vos variables d'environnement dans le fichier osm_rooting_React_Neo4j/.env
```
REACT_APP_NEO4J_URI=XXX
REACT_APP_NEO4J_USER=XXX
REACT_APP_NEO4J_PASSWORD=XXX
REACT_APP_MAPBOX_TOKEN=XXX
```

#Neo4j étant toujours actif, ouvrez une autre fenêtre terminal/console, placez-vous dans le dossier osm_rooting_React_Neo4j.

#puis faites :
```
npm install
npm start
```
