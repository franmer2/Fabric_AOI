# Microsoft Fabric + Azure Open AI

## Prérequis

- Une [souscription](https://azure.microsoft.com/en-ca/free/) Azure avec les services suivants :
  - [Azure SQL database](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview?view=azuresql). **ATTENTION !** La base de données doit être déjà créée. 
    - Suivez la [Procédure pour créer une base Azure SQL DB](https://learn.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart?view=azuresql&tabs=azure-portal).     
  - [Azure AI Search](https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search)
    - Si vous souhaitez utiliser le classement sémantique, il faut configurer au minimum un service avec un tier de base ou standard (S1, S2, S3). 
    - **ATTENTION !!** Il n'est pas possible de changer de tier après le déploiement du service.
    - Si vous souhaitez ajouter des enrichissements par l'intelligence artificielle, Il est possible d'utiliser les services cognitifs d'Azure. Vous pouvez créer un service Azure AI en suivant ce [lien](https://learn.microsoft.com/en-us/azure/ai-services/multi-service-resource?tabs=windows&pivots=azportal). (Je vais utiliser un service existant plus tard dans cet article).
    - **ATTENTION !!!** Au moment où j'écris cet article, les noms des colonnes avec un espace n'étaient pas supportés (j'ai donc dû renommer certaines colonnes).
  - [Azure Open AI](https://learn.microsoft.com/en-us/azure/ai-services/openai/overview)
- Une capacité [Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/enterprise/buy-subscription) :
  - Un espace de travail associé à la capacité Microsoft Fabric. [Documentation](https://learn.microsoft.com/en-us/fabric/get-started/workspaces#license-mode).

## Vue globale

Dans cet article nous allons voir comment préparer les données se trouvant dans un Lakehouse pour les exploiter ensuite avec Azure OpenAI.

Ci-dessous, une vue globale de l'architecture que nous allons mettre en place :

![Architecture](Pictures/001_ArchitectureGenerale.png)


Les fichiers pour cet article se trouvent [ici](/Data/) dans mon repos Github. 
Pour cet exemple, ces fichiers sont stockés dans la partie "File" de mon Lakehouse.

![Lakehouse](Pictures/002.png)

## Création du Dataflow

Vérifiez que vous êtes bien sous le persona "Data Engineering". Puis, à partir d'un espace de travail associé à une capacité Fabric, cliquez sur le bouton "New" puis sélectionnez "dataflow Gen2" :

![DataflowGen2](Pictures/003.png)

À partir de l'interface de Dataflow Gen2, cliquez sur "Get Data", puis sur "More" :

![GetData](Pictures/004.png)

Dans la rubrique "New sources", cliquez sur "View more" :

![ViewMore](Pictures/005.png)

Cliquez sur "Microsoft Fabric" puis sur "Lakehouse" :

![ViewMore](Pictures/006.png)

Définissez la connexion sur votre Lakehouse et cliquez sur "Next" :

![Next](Pictures/007.png)

Sélectionnez les fichiers. Dans cet exemple, je sélectionne des fichiers stockés dans la section "File" de mon lakehouse. Une fois la sélection faîte, cliquez sur "Create" :

![File](Pictures/008.png)

### Nettoyage initial des données

Vous devez donc vous retrouver avec une interface similaire à celle ci-dessous. Il sera peut-être nécessaire d'effectuer des transformations au niveau de chacun des fichiers avant de fusionner les fichiers entre eux.

![File](Pictures/009.png)

Par exemple, au niveau du fichier "product.csv", on remarque que la première ligne semble être l'entête des colonnes. On va donc utiliser la transformation "Use first row as headers" pour refléter la réalité du fichier :

![FirstRow](Pictures/010.png)

Vous devriez obtenir un résultat similaire à celui ci-dessous. Faîtes la même chose avec les autres fichiers si besoin.

![FirstRow](Pictures/011.png)

Dans notre exemple, le fichier "productAddress.csv" semble ne pas avoir été traité correctement. Effectivement, il semble être vu comme un fichier binaire et non comme un fichier csv. En y regardant de plus près, il semblerait que ça soit un problème lié au séparateur utilisé par Microsoft Fabric Dataflow Gen2 lors de la transformation initiale.

![FirstRow](Pictures/012.png)

Directement depuis l'interface, **changez le délimiteur ":" par ","**. Changez aussi le nombre de colonnes que vous souhaitez conserver. Ici j'ai mis 9, car je souhaite conserver toutes les colonnes pour le moment.

![FirstRow](Pictures/013.png)

Ensuite, faîtes les transformations nécessaires pour améliorer la qualité des données (comme rajouter la première ligne comme entête des colonnes 😉)

![FirstRow](Pictures/014.png)

Une fois que tous les fichiers présentent un bon niveau de qualité, on peut commencer à créer des transformations pour créer la vue avec les informations nécessaires pour alimenter notre future base de données. Pour rappel, cette base servira de source de données pour notre service Azure AI Search qui alimentera le service Azure Open AI.

Les étapes suivantes seront des étapes de fusion de données afin d'arriver à notre vue finale.

### Fusion des requêtes

Cliquez sur les 3 petits points, en haut à droite de la boîte de requêtes. Cliquez ensuite sur "Merge queries as new" :

![FirstRow](Pictures/015.png)

Sélectionnez le fichier avec lequel vous souhaitez faire la fusion. Dans cet exemple, je vais prendre le fichier "productCategory.csv". De plus, je décide de faire une jointure de type "Inner" pour être certain d'avoir les données correspondantes venant des 2 fichiers.

Cliquez sur le bouton "Ok":

![FirstRow](Pictures/016.png)

La fusion des données est faîte. Cependant, il vous faut sélectionner les colonnes du fichier de droite que vous souhaitez conserver. Pour ce faire, bougez la barre de défilement vers la droite jusqu'à la colonne portant le nom de votre fichier. Dans mon cas, la colonne se nomme **"productCategory csv"**.

Cliquez sur les flèches pour sélectionner les colonnes à conserver. Cochez les cases pour les colonnes à conserver puis cliquez sur "Ok"

![Fleche](Pictures/017.png)

N'oubliez pas de bien renommer les colonnes afin d'avoir un jeu de donné compréhensible par les utilisateurs métier. **ATTENTION !!! Pour rappel, Azure AI Search ne supporte pas les noms de colonnes avec des espaces !**

![Fleche](Pictures/018.png)

Répétez la procédure de fusion, de nettoyage et de renommage afin d'obtenir une vue avec les informations pertinentes que vous souhaitez livrer à vos utilisateurs métier.

**ATTENTION !!**. Pour certaines requêtes de fusions, choisissez bien le type de jointure. Par exemple, avec le fichier "productOrderDetails csv", il est préférable d'utiliser une jointure de type "Right outer" (si le fichier est à droite de la jointure, bien entendu). Un bon moyen de vérifier si on récupère le bon nombre de lignes, est de vérifier le nombre de lignes retourné en bas de la fenêtre de fusion.

![Fusion](Pictures/019.png)

Voici la vue globale de mon Dataflow Gen2 :

![DataFlowGlobal](Pictures/020.png)

### Définir la destination du Dataflow

Une des nouveautés majeures du Dataflow Gen2, est la possibilité pour chaque requête de définir, si besoin, une destination. Et ça ouvre la porte à de nombreux scénarios, comme l'intégration des données de Microsoft Fabric avec Azure Open AI !

Cliquez sur votre requête finale (Il est possible de le faire pour toutes les requêtes de votre Dataflow), puis cliquez sur le bouton **"Add data destination"**. Plusieurs options s'offrent alors à vous. Dans notre exemple, nous allons choisir "Azure SQL Database".

![DataFlowDestination](Pictures/021.png)

Renseignez les informations de connexions de votre base Azure SQL Database. Cliquez sur "Next" :

![DataFlowDestination](Pictures/022.png)

Donnez un nom à votre table puis cliquez sur "Next" :

![DataFlowDestination](Pictures/023.png)

Définissez ensuite les méthodes de mise à jour. Dans le cas de cet exemple je conserve l'option "Replace". Cliquez sur "Save settings" :

![DataFlowDestination](Pictures/024.png)

Une fois toutes ces étapes terminées, cliquez sur le bouton "Publish" pour exécuter les transformations et créer une nouvelle table dans notre base de données Azure SQL Database.

![DataFlowDestination](Pictures/025.png)

L'ensemble du process devrait prendre quelques minutes. Il est possible de vérifier l'état de la mise à jour du Dataflow depuis l'espace de travail :

![RefreshHistory](Pictures/026.png)

Si tout se passe bien 😊:

![RefreshHistory](Pictures/027.png)

Depuis le [portail Azure](https://portal.azure.com/), connectez-vous à votre serveur Azure SQL DB pour vérifier si la table a bien été créée dans votre base de données Azure SQL DB :

![RefreshHistory](Pictures/028.png)


# Azure AI Search

Maintenant que nous avons une table avec les données souhaitées, nous allons les utiliser pour préparer un index qui sera par la suite utilisé par le service Azure Open AI.

## Création d'un groupe de ressources

Vous pouvez créer un groupe de ressources pour regrouper les ressources que vous allez créer. Dans mon cas, la base de données Azure SQL se trouve dans un autre groupe de ressources, mais dans la même souscription.

Depuis le [portail Azure](https://portal.azure.com/), cliquez sur le bouton "Create a resource".

![Create](Pictures/029.png)

Puis cherchez "Resource group".

![Create](Pictures/030.png)

Remplissez les informations puis cliquez sur "Review + create" et validez la création du groupe de ressources en cliquant sur le bouton "Create".

![Create](Pictures/031.png)

Une fois le groupe de ressources crée, allez dedans en cliquant sur le bouton "Go to resource group".

![Create](Pictures/032.png)

## Création du service Azure AI Search

Dans cette section nous allons faire les étapes suivantes :
- Création d'une source de données
- Création d'un index
- Création d'un indexer


Depuis le groupe de ressources que vous avez créé précédemment, cliquez sur le bouton "Create"

![Create](Pictures/033.png)

Puis chercher le service Azure AI Search.

![Create](Pictures/034.png)

Remplissez les informations et créez le service.

![Create](Pictures/035.png)

Une fois le service créé cliquez sur le bouton "Go to resource".

![Create](Pictures/036.png)

#### Source de données

**ATTENTION !!!** Au moment où j'écris cet article, les noms des colonnes avec un espace n'étaient pas supportés (j'ai donc dû renommer certaines colonnes).

Une fois sur la page **"Overview"** du service Azure AI search, cliquez sur **"Import data"**

![Create](Pictures/037.png)

Sélectionnez la source de données. Ici on va choisir Azure SQL Database.

![Create](Pictures/038.png)

Remplissez les informations de connexion. Le lien **"Choose an existing connection"** permet de retrouver rapidement la chaine de connexion vers notre base Azure SQL.

![Create](Pictures/039.png)

**De manière optionnelle**, il est possible d'enrichir l'index en utilisant les services cognitifs d'Azure, en fonction des documents de la source de données. Il est possible de tester cette fonctionnalité gratuitement mais de manière limitée. Il est aussi possible d'utiliser son propre service Azure AI pour plus de fonctionnalités. Plus d'information sur les fonctionnalités d'enrichissements en suivant [ce lien](https://learn.microsoft.com/en-us/azure/search/cognitive-search-concept-intro).

Il est aussi possible d'utiliser le "Knowledge store" pour conserver le contenu généré par les services Azure AI pour des scénarios en dehors de la recherche (comme la création de rapport Power BI). Plus d'information [ici](https://learn.microsoft.com/en-us/azure/search/knowledge-store-concept-intro?tabs=portal).

Dans cet exemple j'utilise un service que j'ai déployé au préalable. Puis j'ai choisi les options suivantes pour le champ **"Description"** ("Source Data Field") :

- "Extract key phrases"
- "Detect Language"
- "Translate Text"

![Index](Pictures/040.png)

Cliquez sur "Next: Customize target index"

#### Création de l'index

Nous allons maintenant configurer l'index. Pour ce faire il est important d'avoir en tête les définitions suivantes :


- **Retrievable** : champs retournés dans une réponse de requête.
- **Filterable** : champs qui acceptent une expression de filtre.
- **Sortable** : champs qui acceptent une expression "OrderBy".
- **Facetable** : champs utilisés dans une structure de navigation à facettes.
- **Searchable** : champs utilisés dans la recherche en texte intégral. Les chaînes sont       utilisables dans une recherche. Les champs numériques et booléens sont souvent marqués comme ne pouvant pas faire l’objet d’une recherche.

De plus, il faut définir le champ "Key". C'est un champ de type "string" qui représente de manière unique chaque document.

Plus d'information [disponible ici](https://learn.microsoft.com/en-us/azure/search/search-get-started-portal#configure-the-index).

Une fois l'index configuré, cliquez sur **"Next: Create an indexer"**.

![Index](Pictures/041.png)

#### Création de l'indexer

La dernière étape configure et exécute l’indexeur. Cet objet définit un processus exécutable. La source de données, l’index et l’indexeur sont créés à cette étape.

Donnez un nom à votre index, définissez la planification d'exécution (pour cet exemple j'ai choisi "Once"), puis cliquez sur "Submit".

![Indexer](Pictures/042.png)

Pour surveiller la bonne exécution de l'indexer, vous pouvez aller dans le menu "Indexer" et vérifier la progression.

![Indexer](Pictures/043.png)

Si tout se passe bien 😊:

![Indexer](Pictures/044.png)

#### Vérification de l'index

Pour vérifier la pertinence de l'index, allez dans le menu "Indexes", puis cliquez sur le nom de votre index.

![Indexer](Pictures/045.png)

Dans l'onglet "Search explorer", dans le champ "Search", entrez le caractère "*", puis cliquez sur le bouton **"Search"**. Vous pouvez maintenant vérifier les retours de l'index :

![Indexer](Pictures/046.png)



#### Classement sémantique

Dans la Recherche Azure AI, le classement sémantique améliore sensiblement la pertinence de la recherche à l’aide de la compréhension du langage pour reclasser les résultats de la recherche.

Pour activer le classement sémantique, cliquez sur "Semantic ranker", puis choisissez un plan. Pour notre démonstration, le plan "Free" sera suffisant.

![Semantic](Pictures/047.png)

Nous allons maintenant configurer le classement sémantique directement **sur l'index que nous venons de créer plus tôt**.

Cliquez sur l'index créé précedemment.

![Indexe](Pictures/045.png)

Cliquez sur "Semantic configurations", puis sur "Add semantic configuration".

![Semantic](Pictures/048.png)

Renseignez les différents champs. Dans cet exemple, je vais utiliser le champ description pour configurer la recherche sémantique. De plus, le fait d'avoir utilisé l'extraction des mots clefs avec les services IA d'Azure, je vais pouvoir les utiliser pour la section "Keyword fields".

![Semantic](Pictures/049.png)

Une fois la configuration faite cliquez sur le bouton "Save".

![Semantic](Pictures/050.png)


# Azure Open AI

Maintenant que notre index est configuré, nous allons l'utiliser avec le service Azure Open AI.

## Création du service Azure Open AI

Depuis le groupe de ressource créé précédemment, cliquez sur "Create".

![AzureOpenAI](Pictures/051.png)

Recherchez le service "Azure OpenAI" et cliquez sur "Create".

![AzureOpenAI](Pictures/052.png)

Pour choisir la région du service, regardez sur ce [site](https://learn.microsoft.com/en-us/azure/ai-services/openai/quotas-limits#regional-quota-limits).

![AzureOpenAI](Pictures/053.png)

Pour cette démonstration, j'ai sélectionné "All network".

![AzureOpenAI](Pictures/054.png)

Vérifiez vos options de déploiement et cliquez sur "Create".

![AzureOpenAI](Pictures/055.png)

Une fois le service déployé cliquez sur le bouton **"Go to resource"**.

![AzureOpenAI](Pictures/056.png)

## Azure Open AI Studio

Depuis le service Azure OpenAI, dans le menu "Overview", cliquez sur "Go to Azure OpenAI Studio".

![AzureOpenAI](Pictures/057.png)

Avant de commencer, il crée au moins un déploiement. Sur la gauche, cliquez sur "Deployments" puis sur "Create new deployment".

![AzureOpenAI](Pictures/061.png)


Une fois dans le studio, cliquez sur la tuile "Bring your own data".

![AzureOpenAI](Pictures/058.png)

Comme le service vient juste d'être créé, aucun déploiement n'est encore présent. Cliquez sur le bouton "Create new deployment" :

![AzureOpenAI](Pictures/059.png)

Un assistant vous guidera pour définir votre premier déploiement de model. Ici, nous allons déployer le model **"gpt-35-turbo"**. Cliquez sur le bouton "Create" :

![AzureOpenAI](Pictures/060.png)

La fenêtre "Chat playground" est maintenant disponible. Nous allons l'utiliser pour y ajouter nos données. Cliquez sur **"Add your data"** puis sur le bouton **"Add a data source"**.

![AzureOpenAI](Pictures/062.png)

Dans la liste déroulante "Select data source", choisissez "Azure AI Search". Puis, renseignez les informations concernant l'index que l'on a créé précédement. Cliquez sur le bouton "Next".

![AzureOpenAI](Pictures/063.png)

Comme on utilise notre propre index, nous allons définir les champs que l'on souhaite utiliser pour répondre aux questions. Il est possible de fournir plusieurs champs pour le champ "Content Data". Il est nécessaire d'inclure tous les champs qui ont du texte relatif à notre cas d’usage.

Ci-dessous, voici comment j'ai configuré le "mapping" des champs :

![AzureOpenAI](Pictures/064.png)

Comme nous avons configuré notre index pour supporter la recherche sémantique dans la liste déroulante "Search Type", nous pouvons choisir "Semantic". Puis sélectionner l'index que vous souhaitez utiliser.

![AzureOpenAI](Pictures/065.png)

Cliquez sur "Save and close".

![AzureOpenAI](Pictures/066.png)

Vous êtes prêt à dialoguer avec vos données. Posez vos questions dans la partie "Chat session".

Ci-dessous un exemple de conversation. Notez que les réponses sont agrémentées des références.

De plus, il est possible de changer de langue durant la discussion. Dans l'exemple ci-dessous, je commence la conversation en Anglais pour ensuite continuer en Français !

![AzureOpenAI](Pictures/067.png)


### Bon à savoir
Lors de l'étape "Index data field mapping", il est possible de définir un champ de l'index pour "File Name" ainsi que pour "Title" :

![AzureOpenAI](Pictures/069.png)

Cela permet de rajouter des informations pour les références comme illustré ci-dessous :

![AzureOpenAI](Pictures/068.png)

## Déploiement dans une application Web

Une fois votre configuration validée, il est possible de la déployer dans :

- Une "web App"
- Un "Power Virtual Agent Bot"

![AzureOpenAI](Pictures/070.png)

Pour cet article nous allons faire un déploiement dans une "web App". Cliquez sur "A new web app..."

L'assistant "Deploy to a web app" apparaît. Remplissez les informations et cliquez sur "Deploy".
(Il est aussi possible de conserver l'historique du chat en cochant la case "Enable chat history in the web app").

![AzureOpenAI](Pictures/071.png)

Au bout d'une dizaine de minutes, vous aurez une application web avec votre application de chat.

Vous pouvez suivre l'évolution du déploiement de votre "web app" en cliquant sur l'icône des notifications.

![AzureOpenAI](Pictures/072.png)

Une fois votre "web app" déployée, vous pouvez directement l'utiliser comme une application indépendante avec vos propres données.

![AzureOpenAI](Pictures/075.png)