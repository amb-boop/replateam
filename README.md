# Playbook de migration SFCC vers commercetools, Contentstack et Algolia

Date de consultation des docs: 2026-05-07  
Périmètre: Etam e-commerce FR et international, migration depuis Salesforce B2C Commerce / SFCC / Demandware vers commercetools, Contentstack et Algolia.

## 0. Synthèse exécutive PM / COMEX

### Décisions structurantes à prendre en premier

| Sujet | Reco opérationnelle | Pourquoi | Owner |
|---|---|---|---|
| Modèle commerce | 1 projet commercetools Europe, Stores par marché, Channels pour prix et stock, Product Selections si assortiment différencié | Les Stores filtrent langues, pays, distribution/supply channels et assortiments, avec limites à surveiller: 100 distribution channels, 100 supply channels et 100 product selections par Store selon la doc Stores commercetools | PM + dev + intégrateur |
| Produit | 1 Product commercetools = modèle commercial / master SFCC, variants = SKU taille/couleur si le PDP groupe naturellement ces choix | commercetools impose un master variant et limite par défaut à 100 variants par Product. Pour lingerie, valider que les combinaisons couleur x taille x bonnet ne dépassent pas ce seuil | e-merch + dev |
| Prix | Standalone Prices si la volumétrie prix par pays/devise/customer group est élevée; Embedded Prices seulement si le scope reste simple | La doc indique 100 embedded prices max par variant, tandis que Standalone Prices visent les grands volumes et peuvent aller jusqu'à 50 000 prix par variant selon l'overview pricing | dev + intégrateur |
| CMS | Contentstack pour pages, blocs, SEO éditorial, guides, landing campagnes; pas pour stock/prix ni catalogue transactionnel | Contentstack est fait pour contenu structuré et livraison via CDA; CMA sert à gérer le contenu. Séparer strictement commerce et éditorial réduit les conflits d'ownership | e-merch + contenu |
| Search | 1 index Algolia produits par locale et environnement, records au niveau variant couleur ou SKU selon UX, `distinct` sur product ID si records variant | Algolia recommande de choisir la structure record selon l'expérience e-commerce; un index a une stratégie de ranking, les tris passent par replicas | e-merch + dev |
| SEO | Préserver les slugs existants quand possible, sinon mapping 301 exhaustif avant go-live | La perte SEO est le risque business le plus élevé hors tunnel de commande | e-merch + PM |
| Gouvernance | Freeze nomenclature avant premier import: keys CT, UIDs CS, index Algolia, locale codes, slugs | Les corrections tardives coûtent cher: keys et UIDs sont souvent utilisées dans API, exports, mappings, règles et monitoring | PM |

### Points de vigilance

| Risque majeur | Impact | Parade |
|---|---:|---|
| Trop de données dans Algolia | Coût, record too big, pertinence dégradée | N'indexer que les attributs utiles à recherche, facettes, ranking et affichage; contrôler la taille moyenne des records |
| Variants commercetools mal modélisés | PDP, stock, prix et panier instables | Atelier produit Etam: MC, gamme, couleur, taille, bonnet, coupe, lot, pack, cadeaux |
| Contentstack trop libre | Pages incohérentes et dette éditoriale | Modular Blocks cadrés, Global Fields SEO, workflow Draft -> Review -> Approved -> Published |
| Droits trop larges | Incident prod, RGPD, fuite clés | Matrice RBAC par outil, séparation admin/indexing/search-only/preview |
| Cutover sous-estimé | Perte CA, commandes ou SEO | Run parallèle, delta imports, smoke tests J-1/J, rollback DNS/applicatif documenté |

### Cadence recommandée

| Phase | Durée cible | Sorties obligatoires |
|---|---:|---|
| Cadrage data et gouvernance | 3 à 4 semaines | Inventaire SFCC, dictionnaire attributs, conventions, RACI, décisions Stores/indices/locales |
| Modélisation et POC imports | 4 à 6 semaines | Product Types CT, Content Types CS, settings Algolia, scripts d'import, rapport d'écarts |
| Migration pilote FR | 6 à 8 semaines | Catalogue saison pilote, pages clés, search, SEO mapping, tests tunnel |
| Industrialisation international | 6 à 10 semaines | Locales, prix/devises, taxes, shipping, traductions, assortiment par marché |
| Cutover | 2 à 4 semaines | Freeze, delta, bascule, monitoring J+7 |

## 1. Cartographie SFCC -> cible

### Checklist

- Sortir l'inventaire complet des objets SFCC par site et par catalogue.
- Marquer chaque objet: migrer, recréer, archiver, abandonner.
- Définir le format pivot unique: JSONL pour imports API, CSV pour revue métier, XML source conservé pour preuve.
- Identifier les objets à double ownership, notamment promotions, SEO, contenus de campagne et search refinements.
- Créer un registre `sfcc_object_mapping.csv` avec source, cible, méthode d'extraction, owner, fréquence delta.

### Mapping opérationnel

| Objet SFCC | Cible principale | Cible secondaire | Extraction SFCC | Format pivot reco | Owner | Notes / risques |
|---|---|---|---|---|---|---|
| Catalog | commercetools Project + Categories + Products | Algolia products index | Business Manager catalog export XML; OCAPI Data/Shop API si delta | XML source + JSONL CT | dev + intégrateur | Garder l'ID catalogue SFCC comme champ legacy |
| Master Products | commercetools Product | Algolia grouped records | BM catalog XML, OCAPI products | JSONL `products` | e-merch + dev | Vérifier mapping MC/gamme: gamme hors nom produit |
| Variation Products / SKUs | Product Variants | Algolia variant records, InventoryEntry | BM catalog XML, OCAPI | JSONL variants + CSV revue | e-merch | Risque >100 variants par Product dans CT |
| Product custom attributes | Product Type AttributeDefinitions | Algolia attributes si search/facet/ranking | BM system object/custom attributes export, OCAPI Meta API | `attribute_dictionary.xlsx` + JSON schema | e-merch + dev | Typage à figer avant import |
| Categories | CT Categories | Algolia categories/facets, Contentstack pages catégorie éditoriales | BM catalog XML | JSONL categories | e-merch | Arborescence et slug catégorie à préserver |
| Category assignments | Product categories | Algolia hierarchical facets | BM catalog XML | CSV `product_category_assignments.csv` | e-merch | Distinguer primary category, classification, multiples |
| Content Slots | Contentstack Modular Blocks / entries | Algolia banners/rules si search merchandising | BM content export, OCAPI Data API slot configurations | JSON + HTML source | contenu + e-merch | Les slots dynamiques SFCC doivent être remodélisés |
| Content Assets | Contentstack Entries / Assets | CDN/DAM | BM content export, OCAPI content assets | JSON + HTML + asset manifest | contenu | Nettoyer HTML SFCC et scripts inline |
| Page Designer pages | Contentstack Page + Modular Blocks | Frontend routes | BM export / custom job si nécessaire | JSON normalized | contenu + intégrateur | À recréer souvent, pas migrer 1:1 aveuglément |
| Promotions | CT Cart Discounts, Product Discounts, Discount Codes | Algolia Rules pour visibilité search | BM promotions/campaign export, OCAPI Data API | CSV métier + JSON CT | e-merch + dev | Moteurs promo non isomorphes: prévoir analyse règle par règle |
| Coupons | CT Discount Codes | CRM/OMS si contraintes usage | BM coupon export, OCAPI Data API | CSV + JSONL | CRM + dev | RGPD et volumétrie codes uniques |
| Campaigns | CT discounts + Contentstack Releases | Algolia Rules | BM campaign export | CSV planning | e-merch | Recréer la coordination go-live dans CS Releases + jobs CT |
| Customer Groups | CT Customer Groups | Algolia secured filters/persona si besoin | BM customer groups export / OCAPI | CSV + JSONL | CRM + dev | Groupes dynamiques SFCC à clarifier |
| Sites | CT Stores, CS environments/locales, Algolia index/app env | Frontend config | BM site export/custom config | YAML/JSON config | PM + dev | Décision 1 Store par marché recommandée |
| Locales | Project languages CT, CS languages, Algolia index locale | Frontend i18n | BM site/locales export | CSV `locale_matrix.csv` | international | Aligner codes IETF/ISO |
| Users & Roles BM | CT OAuth clients/scopes, CS roles, Algolia team/API keys | SSO/SCIM | BM users/roles export | RACI XLSX | PM + sécurité | Pas migrer les users tels quels |
| Workflows | Contentstack Workflows + Publish Rules | PM process | BM roles/process docs | BPMN/Markdown | PM + contenu | Les workflows SFCC ne se transposent pas automatiquement |
| Search Refinements | Algolia facets/filtering attributes | CT searchable attributes non prioritaire | BM search refinements export | CSV `facet_mapping.csv` | e-merch | Facettes doivent être localisées et ordonnées |
| Sorting Rules | Algolia replicas / customRanking | Frontend sort UI | BM sorting rules export | JSON settings | e-merch + dev | 1 ranking par index, donc replicas |
| URL rules | Frontend router + 301 rules | Contentstack URL field | BM URL rules export, crawl site | CSV redirects | SEO | Décision slug preserve/refonte |
| Redirects | Edge/server 301 | Algolia Rules redirects pour search query | BM redirects, crawl logs, GSC | CSV `redirects_301.csv` | SEO + dev | Éviter d'utiliser Algolia comme seul mécanisme 301 |
| SEO meta | CT product meta, CS Global Field SEO | Algolia display only | BM catalog/content export | CSV + JSON | SEO + contenu | Product SEO dans CT; page SEO dans CS |
| Robot rules | Frontend/hosting config | Aucun | BM site preferences | TXT + config repo | SEO + dev | À versionner |
| Hooks/jobs SFCC | Pipeline intégration, API Extensions, Subscriptions, schedulers | iPaaS | Code review SFCC cartridges/jobs | Architecture Decision Record | dev + intégrateur | Reconcevoir, pas migrer |
| OCAPI/SCAPI configs | API clients CT, frontend services, export scripts | Aucun | BM OCAPI settings | JSON source | dev | Salesforce doc: OCAPI settings pilotent permissions par client/site |

Sources: [Salesforce OCAPI overview](https://developer.salesforce.com/docs/commerce/commerce-api/guide/get-started-with-ocapi.html), [Salesforce content assets](https://developer.salesforce.com/docs/commerce/ocapi/guide/b2c-content-assets-for-developers.html), [Salesforce catalog object import/export](https://help.salesforce.com/s/articleView?id=cc.b2c_catalog_object_import_export.htm&type=5), [SFCC Catalog class](https://developer.salesforce.com/docs/commerce/b2c-commerce/references/b2c-script-api/dw.catalog.Catalog.html).

### Décisions

| Décision | Options | Reco |
|---|---|---|
| Migration contenu SFCC | Migration HTML brute, recréation manuelle, remodélisation semi-automatique | Remodélisation semi-automatique: extraire, classer, nettoyer, recréer en Contentstack |
| Promotions | Migration automatique, analyse règle par règle | Analyse règle par règle avec tests panier |
| Clients et commandes | Migration vivante, archive consultable, double approche | Archive commandes historiques sauf besoin service client; clients à décider avec RGPD/CRM |

### À clarifier

- Export SFCC exact disponible chez Etam: quels jobs déjà présents, quels catalogues, quelles bibliothèques de contenu.
- Droits Etam sur médias CDN SFCC après sortie de contrat.
- Historique commandes: obligation légale, besoin service client, solution archive.

## 2. Modélisation des données

### 2.1 commercetools

Sources clés: [Product Types](https://docs.commercetools.com/api/projects/productTypes), [Products](https://docs.commercetools.com/api/projects/products), [Product catalog overview](https://docs.commercetools.com/api/product-catalog-overview), [Stores](https://docs.commercetools.com/api/projects/stores), [Channels](https://docs.commercetools.com/api/projects/channels), [Product Selections](https://docs.commercetools.com/api/projects/product-selections), [Standalone Prices](https://docs.commercetools.com/api/projects/standalone-prices), [Pricing overview](https://docs.commercetools.com/api/pricing-and-discounts-overview), [Types / Custom Fields](https://docs.commercetools.com/api/projects/types), [Inventory](https://docs.commercetools.com/api/projects/inventory).

#### Checklist

- Construire un dictionnaire attributs SFCC avec typage, localisation, source, usage search/facet/ranking, obligatoire ou non.
- Créer un Product Type minimum viable, puis enrichir par familles, pas par catégories commerciales.
- Valider le seuil 100 variants par Product sur les familles Etam.
- Décider `priceMode`: Embedded vs Standalone.
- Créer Channels: distribution par marché/prix, supply par entrepôt ou zone stock.
- Créer Stores par marché avec langues, pays, channels et product selections.
- Définir Custom Types uniquement pour les champs métier non catalogues: Cart, LineItem, Customer, Order, Store, Category si nécessaire.

#### Product Types proposés

| Product Type | Usage | Attributs produit | Attributs variant | Owner |
|---|---|---|---|---|
| `lingerie` | soutiens-gorge, culottes, bodies, ensembles | `range_key`, `range_label`, `product_name`, `family`, `fit`, `padding`, `wire`, `collection`, `season`, `care`, `composition`, `seo_*` | `color_code`, `color_label`, `size`, `cup`, `ean`, `sku_status`, `launch_date`, `image_set_id` | e-merch |
| `nightwear` | pyjamas, nuisettes, homewear | `range_key`, `product_name`, `material`, `season`, `collection`, `care`, `composition` | `color_code`, `size`, `ean`, `length`, `image_set_id` | e-merch |
| `swimwear` | maillots, beachwear | `range_key`, `product_name`, `shape`, `coverage`, `season`, `composition` | `color_code`, `size`, `cup`, `ean` | e-merch |
| `accessory` | accessoires, lessive, collants si applicable | `product_name`, `family`, `usage`, `composition` | `color_code`, `size`, `ean` | e-merch |

#### Règle Etam gamme vs nom produit

- `product_name`: nom commercial sans gamme.
- `range_key`: identifiant stable de la gamme.
- `range_label`: libellé localisé de la gamme.
- `display_name_front`: construit côté frontend ou pipeline: gamme + nom si l'UX le requiert, mais pas stocké comme source métier unique.
- Risque: si la gamme reste concaténée dans le nom, impossible de filtrer, traduire ou renommer proprement une gamme.

#### AttributDefinition: règles

| Type de donnée | Type CT recommandé | Localisé | Searchable CT | Notes |
|---|---|---:|---:|---|
| Couleur affichée | LocalizedEnum ou LocalizableText | Oui | Oui si recherche CT nécessaire | Algolia reste moteur search front |
| Code couleur | Text / Enum | Non | Non | Clé stable venant PIM/SFCC |
| Taille | Enum | Non | Oui si filtres CT | Prévoir ordre métier hors libellé |
| Bonnet | Enum | Non | Oui si filtres CT | Ne pas mélanger taille et bonnet |
| Composition | LocalizableText | Oui | Non ou oui selon PDP search | Long texte: attention taille searchable |
| Saisonnalité | Enum | Non | Oui | Ranking Algolia possible |
| Best-seller score | Number | Non | Non CT, oui Algolia | Calculé par analytics/ERP |
| SEO title/description produit | LocalizableText | Oui | Non | CT product meta si product page |

Doc à respecter: les noms d'attribut CT font 2 à 256 caractères et acceptent `A-Za-z0-9_-`; `isSearchable` rend l'attribut disponible pour Product Search/Product Projection Search selon le type et le niveau. Les Products ont par défaut 100 variants maximum et les Embedded Prices 100 prix maximum par variant. Voir [Product Types](https://docs.commercetools.com/api/projects/productTypes) et [Products](https://docs.commercetools.com/api/projects/products).

#### Stores, Channels, prix et stock

| Concept | Reco Etam | Décision à trancher |
|---|---|---|
| Store | 1 Store par marché e-commerce: `fr`, `be_fr`, `be_nl`, `de`, `es`, etc. | Confirmer si tous les marchés partagent les mêmes assortiments |
| Distribution Channel | Prix/assortiment commercial par marché ou groupe marché | Channel par pays vs zone de prix |
| Supply Channel | Entrepôt ou source stock: ecom central, magasin, marketplace si applicable | Besoin ship-from-store ou stock marketplace |
| Product Selection | Assortiment par Store si certains marchés excluent des produits | Base catalog + exclusions par marché |
| Standalone Prices | Recommandé si prix par pays/devise/customer group/promos nombreuses | Valider plan et impacts performance/API |

### 2.2 Contentstack

Sources clés: [Modular Blocks](https://www.contentstack.com/docs/developers/create-content-types/modular-blocks), [Global Field](https://www.contentstack.com/docs/developers/create-content-types/global), [Reference Field](https://www.contentstack.com/docs/developers/create-content-types/reference/), [About Branches](https://www.contentstack.com/docs/developers/branches/about-branches), [Environments](https://www.contentstack.com/docs/developers/set-up-environments/about-environments), [Languages](https://www.contentstack.com/docs/developers/multilingual-content/about-languages/), [Fallback Languages](https://www.contentstack.com/docs/developers/multilingual-content/about-fallback-languages/), [Workflows](https://www.contentstack.com/docs/developers/set-up-workflows-and-publish-rules/about-workflows), [Publish Rules](https://www.contentstack.com/docs/developers/set-up-workflows-and-publish-rules/add-a-publish-rule), [Roles](https://www.contentstack.com/docs/developers/invite-users-and-assign-roles/types-of-roles), [CMA](https://www.contentstack.com/docs/developers/apis/content-management-api), [CDA](https://www.contentstack.com/docs/developers/apis/content-delivery-api).

#### Checklist

- Définir `page` comme content type conteneur, avec `seo` Global Field et `components` Modular Blocks.
- Créer des content types autonomes pour contenus réutilisables: FAQ, guide taille, lookbook, menu, legal page.
- Utiliser les références vers commercetools sous forme de champs stables: `product_key`, `sku`, `category_key`, pas de copie prix/stock.
- Utiliser Releases pour campagnes multi-entrées.
- Créer branches `main`, `release_xxx`, `hotfix_xxx` seulement si inclus dans le plan et si l'équipe sait merger.

#### Content Types proposés

| UID | Rôle | Champs clés | Owner |
|---|---|---|---|
| `page_generic` | Page flexible | `title`, `slug`, `seo`, `components`, `market_visibility`, `tracking_tags` | contenu |
| `page_category_editorial` | Landing/category enrichie | `category_key`, `slug`, `seo`, `hero`, `components`, `linked_algolia_rule_context` | e-merch + contenu |
| `block_hero` | Hero campagne | `title`, `subtitle`, `asset_desktop`, `asset_mobile`, `cta`, `theme` | contenu |
| `block_product_carousel` | Carrousel produits | `title`, `source_type`, `product_keys`, `algolia_filter`, `cta` | e-merch |
| `block_editorial_rte` | Texte éditorial | `body_json_rte`, `layout` | contenu |
| `guide_size` | Guide tailles | `body`, `tables`, `product_family`, `locale_scope` | contenu + e-merch |
| `faq` | FAQ | `question`, `answer`, `tags`, `market_visibility` | contenu |
| `lookbook` | Editorial saison | `title`, `assets`, `linked_products`, `components` | contenu |
| `navigation_menu` | Menu | Modular Blocks: internal page, external link, category link, product link | e-merch |
| `redirect_content` | Redirections éditoriales si gérées CMS | `source`, `target`, `status`, `market` | SEO |

#### Limites à intégrer au design

- Contentstack indique 5 Modular Blocks fields max par content type et 100 block definitions par Modular Blocks field dans la page Modular Blocks; la page Field Limitations indique 400 champs max par content type et 500 champs par entrée. À valider avec votre plan car les docs historiques peuvent varier par version de stack.
- Les branches sont plan-based, UID en minuscules, `_` comme séparateur, 15 caractères max, et compare/merge limité aux content types et global fields via CMA/CLI selon la doc Branches.
- Les assets: 700 MB max via UI, 100 MB via API, 10 assets par batch, 10 000 assets par stack par défaut selon Asset Limitations.

### 2.3 Algolia

Sources clés: [Prepare records](https://www.algolia.com/doc/guides/sending-and-managing-data/prepare-your-data), [Algolia records](https://www.algolia.com/doc/guides/sending-and-managing-data/prepare-your-data/in-depth/what-is-in-a-record), [Ecommerce records](https://www.algolia.com/doc/guides/sending-and-managing-data/prepare-your-data/how-to/ecommerce-records/), [Choosing indices](https://www.algolia.com/doc/guides/sending-and-managing-data/prepare-your-data/in-depth/choosing-between-one-or-more-indices), [Replicas](https://www.algolia.com/doc/guides/managing-results/refine-results/sorting/in-depth/replicas), [Rules](https://www.algolia.com/doc/guides/managing-results/rules/rules-overview/), [Faceting](https://www.algolia.com/doc/guides/managing-results/refine-results/faceting), [API keys](https://www.algolia.com/doc/guides/security/api-keys), [Events](https://www.algolia.com/doc/guides/sending-events), [Service limits](https://www.algolia.com/doc/guides/scaling/algolia-service-limits/).

#### Checklist

- Décider record level: SKU, couleur, ou master.
- Créer 1 index produits par locale et environnement.
- Créer indices séparés pour contenu éditorial, query suggestions, éventuellement FAQ.
- Définir `searchableAttributes`, `attributesForFaceting`, `customRanking`, replicas.
- Activer click/conversion/view events dès J1 pour Analytics, Recommend, Personalization, NeuralSearch, Dynamic Re-Ranking.
- Définir qui peut éditer Rules et qui peut pousser settings.

#### Structure d'index recommandée

| Index | Exemple | Record | Usage |
|---|---|---|---|
| Produits FR prod | `etam_fr_products_prod` | 1 record par variant couleur ou SKU selon UX | Search, PLP, navigation |
| Produit tri prix asc | `etam_fr_products_prod_price_asc` | Replica | Tri prix croissant |
| Produit tri prix desc | `etam_fr_products_prod_price_desc` | Replica | Tri prix décroissant |
| Produit nouveauté | `etam_fr_products_prod_newness_desc` | Replica | Tri nouveautés |
| Query Suggestions | `etam_fr_query_suggestions_prod` | Query | Autocomplete |
| Contenu éditorial | `etam_fr_content_prod` | 1 record par page/bloc indexable | Federated search |

#### Record produit type

```json
{
  "objectID": "fr-FR:product-key:sku-or-color",
  "productID": "ct-product-key",
  "sku": "sku",
  "name": "Nom sans gamme",
  "range": "Gamme",
  "categoryPaths": ["Lingerie > Soutiens-gorge"],
  "color": "Noir",
  "size": ["85B", "90B"],
  "price": 39.99,
  "currency": "EUR",
  "inStock": true,
  "newness": 1716508800,
  "salesRank": 124,
  "marginRank": 87,
  "image": "https://...",
  "url": "/fr_FR/p/...",
  "_tags": ["market:fr", "season:pe26"]
}
```

Reco: pour Etam, partir sur records variant couleur avec tailles disponibles en tableau si les PLP affichent surtout des coloris; passer au record SKU si les filtres taille/bonnet et disponibilité doivent être exacts au hit. Utiliser `distinct` sur `productID` si records SKU et UX groupée.

## 3. Nomenclature et conventions à figer

### Checklist

- Publier un fichier `naming_conventions.md`.
- Bloquer tout import tant que les clés Product Type, Category, Attribute, Store, Channel, Content Type UID, index Algolia ne sont pas validées.
- Mettre un validateur CI sur les fichiers de mapping.

### Conventions

| Domaine | Convention | Exemple | Règle doc / risque |
|---|---|---|---|
| CT keys | lowercase kebab ou snake, 2-256 chars, `A-Za-z0-9_-` | `lingerie-sg`, `store-fr` | Les keys CT suivent le pattern documenté sur Stores/Types/Product Types |
| CT attribute name | snake_case lisible, stable | `range_key`, `color_code`, `cup_size` | Nom 2-256, `A-Za-z0-9_-`; ne pas renommer sans migration |
| CS content type UID | snake_case, domaine préfixé | `page_category_editorial`, `block_hero` | Contentstack accepte alphanum + underscores, UID content type max 200 |
| CS branch UID | lowercase + `_`, max 15 | `release_pe26`, `hotfix_seo` | Limite Branches |
| Algolia index | `{brand}_{locale}_{type}_{env}` | `etam_fr_products_prod` | Index names publics dans les requêtes: pas de PII |
| Algolia replica | `{primary}_{sort}_{direction}` | `etam_fr_products_prod_price_asc` | Aligné doc replicas |
| Locale | IETF cohérent entre outils | `fr-FR`, `en-GB`, `de-DE` | CT Stores utilise languages du Project; CS languages; Algolia par locale |
| Slug | lowercase, hyphen, sans ID technique si possible | `/soutiens-gorge/ampli/` | CT product slug pattern documenté: `A-Za-z0-9_-`, 2-256, unique par projet |
| Releases CS | `{market}_{campaign}_{date}` | `fr_soldes_2026_06_24` | Facile à auditer |
| Tags CS/Algolia | `domain:value` | `season:pe26`, `market:fr` | Évite collisions |

### Décisions

| Sujet | Options | Reco |
|---|---|---|
| snake_case vs camelCase | snake_case partout métier, camelCase seulement si SDK impose | snake_case pour CS/CSV/Algolia; CT accepte `_` |
| Codes marchés | `fr`, `fr-FR`, `FR` | `fr-FR` pour locale, `fr` pour Store/index segment |
| Clés catégories | importer SFCC IDs ou normaliser | conserver `legacy_sfcc_id`, créer `category_key` stable normalisé |

## 4. Extractions SFCC à produire

### Checklist générale

- Produire un export initial complet.
- Produire un delta quotidien pendant build.
- Produire un delta freeze J-1/J.
- Stocker les exports sources immuables dans un répertoire daté.
- Associer chaque export à un contrôle qualité: nombre, champs obligatoires, doublons, valeurs inconnues.

| Export | Méthode | Format source | Format pivot | Fréquence | Destinataire | Contrôles |
|---|---|---|---|---|---|---|
| Produits masters | BM catalog export XML, OCAPI Data/Shop si delta | XML/JSON | JSONL CT + CSV revue | Initial + delta | intégrateur | count, ID unique, nom, catégorie |
| Variations/SKUs | BM catalog XML | XML | JSONL variants | Initial + delta | intégrateur | SKU unique, EAN, taille/couleur |
| Attributs custom | BM metadata export, OCAPI Meta API | XML/JSON | `attribute_dictionary.xlsx` | Initial + changement | e-merch/dev | type, valeurs enum, localisé |
| Prix | BM price books export | XML/CSV | JSONL standalone prices | Initial + delta quotidien | dev | currency, country/channel, customer group |
| Stock | OMS/WMS source prioritaire ou SFCC inventory lists | CSV/XML | JSONL inventory | Delta fréquent | dev | SKU, supply channel, quantity |
| Catégories | BM catalog XML | XML | JSONL categories | Initial + delta | e-merch | parent, slug, online flag |
| Assignations produit-catégorie | BM catalog XML | XML | CSV | Initial + delta | e-merch | produits sans catégorie |
| Médias | URL CDN + extraction binaires si droits | CSV + fichiers | asset manifest | Initial + delta | contenu | 404, poids, dimensions |
| Content Assets | BM content export, OCAPI Data content | XML/HTML | JSON + HTML nettoyé | Initial + delta | contenu | HTML valide, dépendances assets |
| Content Slots | BM site export, OCAPI Data slot configs | XML/JSON | JSON mapping | Initial + delta | e-merch | règle slot, calendrier |
| Page Designer | BM export/custom job | JSON/XML | JSON normalized | Initial | contenu/dev | composants non supportés |
| Promotions | BM export, OCAPI Data | XML/JSON | CSV métier + JSON CT | Initial + freeze | e-merch/dev | conditions, cumul, calendrier |
| Coupons | BM export | CSV/XML | CSV | Initial + freeze | CRM/dev | codes uniques, dates, usages |
| Customer Groups | BM export/OCAPI | JSON/CSV | JSONL CT | Initial + delta | CRM | dynamiques vs statiques |
| SEO meta | catalog/content exports + crawl | CSV | CSV | Initial + freeze | SEO | title/description manquants |
| Redirects | BM redirects + crawl/logs/GSC | CSV | CSV 301 | Initial + freeze | SEO/dev | boucle, chaines, 404 |
| Sitemap | SFCC sitemap | XML | CSV URL inventory | Initial + J-1 | SEO | coverage |
| Robots | BM/site config | TXT | TXT | Initial | SEO/dev | directives par env |
| Clients | OCAPI/Data API ou export client | CSV/JSON | à définir RGPD | Décision | CRM/legal | consentements, hashes |
| Commandes | Export order history ou archive BI | CSV/JSON | archive/search | Décision | CS/finance | obligations légales |
| Jobs/hooks/code | repo cartridges + BM jobs | code/config | ADR + inventory | Initial | dev | dépendance tiers |

### Artefacts

- `exports/source/YYYYMMDD/sfcc_catalog.xml`
- `exports/source/YYYYMMDD/sfcc_price_books.zip`
- `exports/pivot/products.jsonl`
- `exports/pivot/product_variants.jsonl`
- `exports/pivot/attribute_dictionary.xlsx`
- `exports/pivot/category_tree.csv`
- `exports/pivot/redirects_301.csv`
- `exports/pivot/content_inventory.csv`
- `exports/qc/export_qc_report.md`

## 5. Users, rôles, droits

### Checklist

- Exporter les rôles BM actuels et lister les utilisateurs réellement actifs.
- Supprimer les comptes nominaux inutiles avant migration.
- Créer une matrice d'accès par outil, environnement, marché, langue.
- Séparer les clés machine des users humains.
- Prévoir SSO/SAML et SCIM si inclus contrats.

### Matrice cible

| Profil | commercetools | Contentstack | Algolia | Owner validation |
|---|---|---|---|---|
| e-merch FR | Merchant Center lecture catalogue/prix, pas API admin | Content Manager FR, publier prod si workflow approved | Merchandising Rules sur index FR, pas settings | PM + e-merch |
| e-merch inter | Accès Stores marchés, lecture prix/stock | Content Manager limité langues/marchés | Rules marchés, synonyms locaux | international |
| Marketing/contenu | Pas accès commerce sauf lecture | Création pages/blocs/assets, workflow review | Accès banners/rules si formé | contenu |
| CRM | Customer Groups lecture/gestion selon besoin | Contenus CRM si stack concernée | Secured API keys/persona si besoin | CRM |
| Customer service | Lecture commandes/client selon outil service | Aucun ou lecture FAQ | Aucun | CS + legal |
| Dev interne | API clients limités par env, manage scopes non prod | Developer non prod, tokens | settings/indexing non prod | tech lead |
| Intégrateur | Accès projet non prod, prod temporaire contrôlé | Developer branches/dev/preprod | Indexing/settings non prod | PM + sécurité |
| Admin sécurité | Gestion API clients/scopes | Owner/Admin restreint | App owner/admin | sécurité |

### Règles par outil

| Outil | Principe | Doc |
|---|---|---|
| commercetools | Créer des API clients par service avec OAuth scopes minimum: `view_products`, `manage_products`, `manage_orders`, etc. | [Scopes](https://docs.commercetools.com/api/scopes), [Authorization](https://docs.commercetools.com/api/authorization) |
| Contentstack | Utiliser rôles système seulement si suffisants, sinon Custom Roles par content type/env/langue. Delivery token = lecture published env; Management token = lecture/écriture CMA. | [Roles](https://www.contentstack.com/docs/developers/invite-users-and-assign-roles/types-of-roles), [CMA tokens](https://www.contentstack.com/docs/developers/apis/content-management-api) |
| Algolia | Ne jamais exposer Admin API key; frontend avec search-only ou secured API keys; clés indexing limitées aux index nécessaires. | [API keys](https://www.algolia.com/doc/guides/security/api-keys) |

### À clarifier

- Contrat Contentstack: disponibilité SSO, SCIM, workflows, branches, nombre de stacks/branches.
- Contrat Algolia: accès Merchandising Studio, Personalization, Recommend, NeuralSearch, virtual replicas.
- Organisation commercetools: teams/projects/business units activés ou non.

## 6. Workflows et gouvernance contenu

### Checklist

- Mettre en place workflow CS: Draft -> Review -> Approved -> Published.
- Créer publish rules prod: publication seulement si stage Approved.
- Créer releases pour campagnes: Soldes, Saint-Valentin, lancement collection.
- Documenter le process catalogue CT: source, contrôles, import, validation, publication.
- Documenter le process Algolia Rules: création, preview, validation, push prod.

### Process cible

| Domaine | Process | Artefacts | Owner |
|---|---|---|---|
| Contenu Contentstack | Création Draft, assignation reviewer, validation locale, publish preprod, release prod | `content_workflow.md`, release checklist | contenu + e-merch |
| Campagne | CS Release + CT discounts + Algolia Rules + frontend flags | `campaign_go_live_sheet.xlsx` | PM |
| Catalogue CT | Import PIM/SFCC delta, validation Product Types, publication product projections, sync Algolia | `catalog_import_runbook.md` | dev + e-merch |
| Algolia | Rule Visual Editor pour e-merch, API/settings par dev, export settings versionné | `algolia_settings/*.json` | e-merch + dev |
| Saison Etam | Batch enrichissement 3000 MC: mapping gamme, attributs, médias, traduction, prix, stock, search | `season_readiness_dashboard.xlsx` | e-merch |

Sources: Contentstack Workflows permettent stages, assignments, approval rules, prevent self-advancement, branch-specific workflows si inclus plan. Publish Rules gouvernent publication/unpublication par branches/environnements. Algolia Rules Visual Editor couvre promote/hide, boost/bury, banners, facet merchandising, contexte device/segment/date selon la doc Rules.

## 7. Internationalisation

### Checklist

- Créer une matrice locale: marché, langue, fallback, devise, taxes, shipping, Store CT, index Algolia, environnement CS.
- Décider où vit chaque traduction: CT, CS ou Algolia calculé.
- Définir fallback explicite et règles de contenu non localisable.
- Tester marchés partiellement traduits.

| Donnée | Source de vérité | Publication |
|---|---|---|
| Nom produit, descriptions, composition | CT attributs localisés | CT -> Algolia locale |
| Contenu page, hero, FAQ, guide tailles | Contentstack entry localisée | CDA locale |
| Searchable product text | Algolia index par locale | Pipeline indexation |
| Prix/devise | CT Price/Standalone Price | Store/channel |
| Taxes | CT Tax Categories + Zones | Cart/order |
| Shipping | CT Shipping Methods + Zones | Cart/order |
| Synonymes | Algolia par index locale | Dashboard/API |

### Stores CT

Reco: 1 Store par marché si le pays a au moins une différence de langue, assortiment, prix, taxe, livraison ou stock. Un projet par région uniquement si contraintes fortes de séparation données, performance, organisation ou contrat. À clarifier avec commercetools pour limites projet et gouvernance Etam.

### Contentstack

- Master language impossible à changer après création de stack selon la doc Set Master Language: choisir avec prudence.
- Fallback language peut hériter du parent puis master si absent.
- Non-localizable fields utiles pour IDs, product keys, tracking, mais plan-based dans certains cas imbriqués.

## 8. Search et merchandising Algolia

### Checklist

- Exporter Search Refinements et Sorting Rules SFCC.
- Définir facettes front: catégorie, taille, bonnet, couleur, forme, prix, collection, matière, promo, nouveauté, stock.
- Mapper chaque refinement SFCC vers attribut Algolia.
- Créer replicas pour tris.
- Créer rules de merchandising pour top catégories et campagnes.
- Câbler events Insights dès le premier test front.

### Settings produits FR type

```json
{
  "searchableAttributes": [
    "unordered(name)",
    "unordered(range)",
    "unordered(categoryNames)",
    "unordered(color)",
    "unordered(material)",
    "sku"
  ],
  "attributesForFaceting": [
    "searchable(categoryPaths)",
    "searchable(color)",
    "size",
    "cup",
    "family",
    "collection",
    "inStock",
    "priceBucket",
    "market"
  ],
  "customRanking": [
    "desc(inStock)",
    "desc(salesRank)",
    "desc(newness)",
    "desc(marginRank)"
  ],
  "replicas": [
    "etam_fr_products_prod_price_asc",
    "etam_fr_products_prod_price_desc",
    "etam_fr_products_prod_newness_desc"
  ]
}
```

### Rules vs Personalization vs Recommend vs NeuralSearch

| Besoin | Fonction Algolia | Reco |
|---|---|---|
| Pinning produits sur requête/catégorie | Rules Visual Editor | Oui J1 |
| Banners search/PLP | Rules banners | Oui si frontend InstantSearch compatible ou rendu custom |
| Redirection requête "carte cadeau" | Rules redirect | Oui pour search; pas pour SEO 301 global |
| Boost saisonnier "soldes" | Rules + customRanking | Oui avec dates et tags |
| Recos PDP/panier | Recommend | Phase 2 si events fiables |
| Personnalisation par affinité | Personalization | Phase 2, après consentement et events |
| NeuralSearch / AI Search | NeuralSearch | Pilote contrôlé après base keyword solide |
| Query Suggestions | Query Suggestions index séparé | Oui J1 search UX |

### Synonymes et dictionnaires

- Créer `synonyms_fr.json`: `culotte`, `shorty`, `tanga`; `soutien gorge`, `soutien-gorge`, `sg`; fautes fréquentes issues analytics SFCC.
- Séparer synonymes réguliers, one-way, corrections.
- Ne pas créer de synonymes qui détruisent le merchandising: `string` vs `tanga` peut être sensible selon catégories.

### Limites à surveiller

- Records comptés sur tous les indices, replicas inclus pour Standard replicas selon support Algolia.
- Taille record selon plan: docs support indiquent 10 KB Build, 100 KB Standard/Premium/Grow avec moyenne 10 KB.
- Service limits publics: 1 000 indices Premium, 50 Grow, 10 Build; 20 virtual replicas par index; 10 000 synonyms par index ou 1 000 Build; conditions/rules et filters limités selon doc Service Limits.
- À clarifier avec Algolia: plan Etam, over-quota, nombre d'applications, virtual replicas, NeuralSearch pricing/quotas.

## 9. SEO et migration URLs

### Checklist

- Crawler SFCC complet: URLs indexables, status, canonical, hreflang, meta, structured data.
- Exporter redirects SFCC.
- Définir stratégie slug: preserve par défaut.
- Produire `redirects_301.csv` source, target, reason, priority, owner.
- Générer sitemaps nouvelle stack et comparer coverage.
- Mettre monitoring J: GSC, logs 404/500, ranking, revenue SEO.

| Élément | Source SFCC | Cible | Owner |
|---|---|---|---|
| Product URL | SFCC URL rules/catalog | Frontend route + CT slug | SEO + dev |
| Category URL | SFCC category slug | Frontend route + CT/CS category key | SEO |
| Editorial URL | Content Assets/Page Designer | CS `slug` | contenu |
| Redirects | BM redirects + crawl + GSC | Edge/server rules | SEO + dev |
| Canonical | Templates SFCC | Frontend | SEO + dev |
| Hreflang | Sitemap/templates | Frontend locale router | international + SEO |
| Structured data | SFCC templates | Frontend Product/BreadcrumbList | dev + SEO |
| Robots | BM preferences | Hosting config | SEO |

### Décision

Reco: Algolia Rules redirect sert aux requêtes internes de recherche, pas comme mécanisme principal de 301 SEO. Les 301 doivent vivre au niveau edge/server/front afin que les robots et liens externes soient correctement servis.

## 10. Médias et DAM

### Checklist

- Inventorier tous les médias SFCC: product images, swatches, content assets, Page Designer.
- Valider droits et durée d'accès au CDN SFCC.
- Décider DAM cible: Contentstack Assets, DAM externe, CDN frontend.
- Créer un asset manifest: legacy URL, filename, checksum, dimensions, target asset UID/URL, owner.
- Générer variantes responsive et formats modernes côté pipeline/CDN.

| Option | Avantages | Risques | Reco |
|---|---|---|---|
| Garder URLs SFCC | Rapide | Dépendance contractuelle, risque 404 post-exit | Non sauf transition courte |
| Contentstack Assets | Simple pour éditorial | Limites stack/assets, pas DAM produit massif si volumétrie très élevée | Oui pour éditorial |
| DAM externe | Gouvernance média produit | Projet supplémentaire | Recommandé si Etam a DAM groupe |
| CDN frontend | Performance | Besoin pipeline image | Oui pour produit si DAM/source PIM existe |

Contentstack Asset Limitations publiques: max 700 MB via UI, 100 MB via API, 10 assets par batch, 10 000 assets par stack par défaut, image input max 50 MB pour optimisation.

## 11. Intégrations et flux tiers

### Checklist

- Lister tous les jobs SFCC et cartridges tiers.
- Pour chaque flux, identifier trigger, fréquence, sens, format, SLA, owner.
- Choisir mécanisme cible: CT API, Import API, API Extensions, Subscriptions, Contentstack webhooks, Algolia API.
- Définir replay/idempotence.

| Flux | SFCC actuel | Cible | Mécanisme recommandé | Owner |
|---|---|---|---|---|
| PIM/catalogue | XML Demandware | CT Products/Product Types | Import API + validation | dev + intégrateur |
| Prix | Price books | CT Standalone/Embedded Prices | Import API price | dev |
| Stock | Inventory lists/job | CT InventoryEntry | API/import delta | dev |
| Orders -> OMS/ERP | SFCC order export job | CT Orders | Subscriptions ou polling API | dev |
| Paiement | cartridge SFCC | Frontend + PSP + CT payment refs | custom service | dev |
| Fraude | cartridge/job | Checkout/order service | API extension/subscription selon timing | dev |
| CRM/emailing | SFCC customer/order feeds | CRM direct + CT events | Subscriptions/iPaaS | CRM |
| Avis clients | SFCC cartridge/feed | Frontend + CT/Algolia display | custom integration | e-merch |
| Fidélité | SFCC customer profile/custom attrs | CT Customer custom fields + CRM | API | CRM |
| Livraison | SFCC shipping methods | CT Shipping Methods/Zones | API config + OMS | dev |
| Search indexing | SFCC -> Algolia connector si existant | CT/CS -> Algolia | commercetools connector ou pipeline custom | dev + Algolia |
| Content publish | BM content | CS webhooks -> build/revalidate | Webhooks/Launch | dev + contenu |

Sources: [commercetools Subscriptions](https://docs.commercetools.com/api/projects/subscriptions), [API Extensions limits](https://docs.commercetools.com/api/limits), [Contentstack webhooks](https://www.contentstack.com/docs/developers/webhooks), [Algolia send/update data](https://www.algolia.com/doc/guides/sending-and-managing-data/send-and-update-your-data/).

## 12. Environnements, branches, releases

### Checklist

- Définir un mapping env unique.
- Séparer données test, données masquées, données prod.
- Créer seed minimal: catégories, 100 produits, 10 pages, 5 promos, 3 marchés.
- Prévoir reset non-prod.

| Environnement | commercetools | Contentstack | Algolia | Usage |
|---|---|---|---|---|
| Dev | Project sandbox dev | Branch `main` dev stack ou env `dev` | App/index dev | Build |
| Preprod | Project preprod | Environment `preprod`, branch release si modèle change | App/index preprod | Recette |
| Prod | Project prod | Environment `prod`, branch main | App/index prod | Live |
| Training | Project sandbox limité | Stack/env training | Index réduit | Formation |

### Reco

- Algolia: idéalement applications séparées par env pour éviter pollution Analytics et accident d'indexation. La doc recommande des indices séparés ou apps/env séparés pour staging/prod.
- Contentstack: utiliser Environments pour destinations de publication; Branches pour changements de modèle ou gros chantiers, pas pour chaque campagne métier.
- commercetools: projet prod distinct; sandbox pour tests; données client masquées hors prod.

## 13. Cutover / go-live

### Checklist run parallèle

- Périmètre pilote: FR desktop/mobile, catalogue complet ou famille pilote selon planning.
- Comparer SFCC vs nouvelle stack: count produits, prix, stock, catégories, URLs, search top queries.
- Geler les changements non essentiels avant bascule.
- Préparer rollback DNS/applicatif et réouverture SFCC si nécessaire.

### Planning

| Moment | Actions | Owner |
|---|---|---|
| J-30 | Validation modèle data, premiers imports complets, SEO crawl baseline, RACI signé | PM |
| J-21 | NRT front, search relevance benchmark, tunnel panier/checkout, monitoring en place | dev + e-merch |
| J-14 | Formation e-merch/contenu, dry run import delta, dry run release campagne | PM |
| J-7 | Freeze conventions, derniers changements structurants interdits, plan rollback validé | PM + dev |
| J-3 | Export full SFCC, delta rehearsal, validation prix/stock/promos | dev + e-merch |
| J-1 | Freeze contenu/catalogue/promo selon fenêtre, delta final, sitemap final, 301 final | PM |
| J | Bascule DNS/routing, smoke tests, monitoring logs/search/orders, décision go/no-go | war room |
| J+1 | Analyse 404, conversion, search no-results, orders, incidents P1 | PM + dev |
| J+7 | SEO/GSC, revenue par marché, backlog stabilisation, réouverture changements | PM |

### Smoke tests minimum

- Home, PLP, PDP, search, no-results, add-to-cart, promo code, login, guest checkout, payment, order confirmation.
- Locale switch, prix/devise, stock, livraison, taxes.
- Contentstack preview/prod publish.
- Algolia query top 100 SFCC, synonyms, facets, sort replicas, rules.
- Redirections top SEO URLs.

## 14. Formation et conduite du changement

### Checklist

- Créer sandbox training avec fausses données réalistes.
- Former par profil, pas par outil seulement.
- Produire runbooks courts: créer page, créer campagne, modifier merchandising, valider produit, analyser no-results.
- Mettre en place support hypercare J+14.

| Profil | Modules | Artefacts |
|---|---|---|
| e-merch FR | CT lecture catalogue, Algolia Rules, Contentstack category pages, campagne | fiches 1 page + exercices |
| international | Localisation CS, indexes locale, QA marché | locale checklist |
| contenu | Modular Blocks, assets, workflows, releases | content playbook |
| CRM | customer groups, codes promo, events | CRM integration sheet |
| dev/intégrateur | imports, API scopes, webhooks, subscriptions, monitoring | runbooks techniques |
| PM | cutover, RACI, risks, release calendar | dashboard projet |

## 15. Risques et dépendances

### Top risques

| Risque | Probabilité | Impact | Mitigation | Owner |
|---|---:|---:|---|---|
| Modèle variant CT incorrect | Haute | Élevé | Atelier produit + tests familles extrêmes | e-merch + dev |
| Promotions non équivalentes SFCC/CT | Haute | Élevé | Catalogue de règles, tests panier automatisés | e-merch + dev |
| SEO URLs mal redirigées | Moyenne | Très élevé | Crawl, mapping 301, monitoring J+7 | SEO |
| Algolia record trop lourd/coûteux | Moyenne | Élevé | Payload minimal, audit taille, plan vendor | dev |
| Droits trop ouverts | Moyenne | Élevé | RBAC, SSO, API keys limitées | sécurité |
| Traductions/fallback incohérents | Moyenne | Moyen | Locale matrix, QA marché | international |
| Médias 404 post-SFCC | Moyenne | Élevé | Réhébergement, manifest, tests 404 | contenu + dev |
| Import delta instable | Haute | Élevé | Idempotence, logs, replay, dry runs | intégrateur |
| Workflows trop complexes | Moyenne | Moyen | Workflow minimal J1, évolution après go-live | PM |
| Dépendance intégrateur | Moyenne | Élevé | Ownership Etam des mappings/scripts/runbooks | PM |

### Dépendances bloquantes

- Contrats vendor signés et plans connus: Contentstack branches/workflows/assets, Algolia replicas/NeuralSearch/record quota, commercetools limits.
- Accès SFCC export complet et droits OCAPI/Data API.
- Inventaire PIM/ERP/OMS/CRM disponible.
- Décision RGPD clients/commandes.
- Équipe e-merch disponible pour arbitrages attributs, promotions, search.

## 16. Annexes

### 16.1 Glossaire

| SFCC | commercetools | Contentstack | Algolia |
|---|---|---|---|
| Master Product | Product avec master variant + variants | Référence `product_key` | `productID` / distinct group |
| Variation Product | Product Variant | Référence SKU | Record SKU/variant |
| Product custom attribute | AttributeDefinition | Champ uniquement si contenu éditorial | Record attribute |
| Category | Category | Page category editorial | Facet/category path |
| Content Asset | Entry / Asset | Entry / Asset | Content record si search |
| Content Slot | Modular Block / placement | Modular Blocks | Rule banner si search |
| Page Designer | Page + Modular Blocks | Page content type | Content record |
| Search Refinement | Non frontal direct | Non | Facet |
| Sorting Rule | Product Search/Projection possible | Non | Replica/customRanking |
| Campaign | Discounts + schedule | Release | Rule date/context |
| Coupon | DiscountCode | Non | Non sauf search display |
| Customer Group | CustomerGroup | Non | secured filters/persona si besoin |
| Site | Store | Environment/locale | Index/app env |
| BM Role | OAuth scopes / org role | Role/Custom Role | API key/team permission |

### 16.2 Templates

#### `attribute_dictionary.csv`

```csv
legacy_sfcc_id,target_tool,target_name,target_type,localized,required,searchable,facetable,ranking,owner,notes
custom.gamme,CT,range_key,text,false,true,false,false,false,e-merch,source of truth gamme
custom.colorLabel,CT,color_label,lenum,true,true,true,true,false,e-merch,localized enum
custom.salesRank,Algolia,salesRank,number,false,false,false,false,true,e-merch,computed daily
```

#### `redirects_301.csv`

```csv
source_url,target_url,status,market,reason,priority,validated
/fr_FR/old-product.html,/fr-FR/p/new-product,301,fr,product slug migration,high,false
```

#### Product Type CT skeleton

```json
{
  "key": "lingerie",
  "name": "Lingerie",
  "description": "Etam lingerie products",
  "attributes": [
    {
      "name": "range_key",
      "label": { "fr-FR": "Gamme" },
      "type": { "name": "text" },
      "isRequired": true,
      "attributeConstraint": "SameForAll",
      "isSearchable": false
    }
  ]
}
```

#### Content Type CS skeleton

```json
{
  "content_type": {
    "title": "Page Category Editorial",
    "uid": "page_category_editorial",
    "schema": [
      { "display_name": "Title", "uid": "title", "data_type": "text", "mandatory": true },
      { "display_name": "Slug", "uid": "slug", "data_type": "text", "mandatory": true },
      { "display_name": "Category Key", "uid": "category_key", "data_type": "text", "mandatory": true }
    ]
  }
}
```

#### Algolia settings skeleton

```json
{
  "indexName": "etam_fr_products_prod",
  "settings": {
    "searchableAttributes": ["unordered(name)", "unordered(range)", "unordered(categoryNames)", "sku"],
    "attributesForFaceting": ["searchable(categoryPaths)", "color", "size", "cup", "family", "inStock"],
    "customRanking": ["desc(inStock)", "desc(salesRank)", "desc(newness)"],
    "replicas": ["etam_fr_products_prod_price_asc", "etam_fr_products_prod_price_desc"]
  }
}
```

### 16.3 Liens directs consultés

#### commercetools

- [Limits](https://docs.commercetools.com/api/limits)
- [Product Types](https://docs.commercetools.com/api/projects/productTypes)
- [Products](https://docs.commercetools.com/api/projects/products)
- [Product catalog overview](https://docs.commercetools.com/api/product-catalog-overview)
- [Stores](https://docs.commercetools.com/api/projects/stores)
- [Channels](https://docs.commercetools.com/api/projects/channels)
- [Product Selections](https://docs.commercetools.com/api/projects/product-selections)
- [Standalone Prices](https://docs.commercetools.com/api/projects/standalone-prices)
- [Pricing and discounts overview](https://docs.commercetools.com/api/pricing-and-discounts-overview)
- [Inventory](https://docs.commercetools.com/api/projects/inventory)
- [Types / Custom Fields](https://docs.commercetools.com/api/projects/types)
- [Custom Fields](https://docs.commercetools.com/api/projects/custom-fields)
- [Carts and Orders overview](https://docs.commercetools.com/api/carts-orders-overview)
- [Subscriptions](https://docs.commercetools.com/api/projects/subscriptions)
- [Scopes](https://docs.commercetools.com/api/scopes)
- [Authorization](https://docs.commercetools.com/api/authorization)
- [Import Containers](https://docs.commercetools.com/api/import-export/import-container)
- [Import Requests](https://docs.commercetools.com/api/import-export/import-requests)

#### Contentstack

- [Create Content Types](https://www.contentstack.com/docs/developers/create-content-types)
- [Modular Blocks](https://www.contentstack.com/docs/developers/create-content-types/modular-blocks)
- [Global Field](https://www.contentstack.com/docs/developers/create-content-types/global)
- [Reference Field](https://www.contentstack.com/docs/developers/create-content-types/reference/)
- [Field Limitations](https://www.contentstack.com/docs/developers/create-content-types/field-limitations)
- [Limitations of Content Types](https://www.contentstack.com/docs/developers/create-content-types/limitations-of-content-types)
- [Branches](https://www.contentstack.com/docs/developers/branches/about-branches)
- [Limitations of Branches](https://www.contentstack.com/docs/developers/branches/limitations-of-branches)
- [Compare and Merge Branches](https://www.contentstack.com/docs/developers/branches/about-compare-and-merge-branches)
- [Environments](https://www.contentstack.com/docs/developers/set-up-environments/about-environments)
- [Languages](https://www.contentstack.com/docs/developers/multilingual-content/about-languages/)
- [Master Language](https://www.contentstack.com/docs/developers/multilingual-content/set-the-master-language)
- [Fallback Languages](https://www.contentstack.com/docs/developers/multilingual-content/about-fallback-languages/)
- [Workflows](https://www.contentstack.com/docs/developers/set-up-workflows-and-publish-rules/about-workflows)
- [Publish Rules](https://www.contentstack.com/docs/developers/set-up-workflows-and-publish-rules/add-a-publish-rule)
- [Roles](https://www.contentstack.com/docs/developers/invite-users-and-assign-roles/types-of-roles)
- [CMA](https://www.contentstack.com/docs/developers/apis/content-management-api)
- [CDA](https://www.contentstack.com/docs/developers/apis/content-delivery-api)
- [Asset Limitations](https://www.contentstack.com/docs/content-managers/working-with-assets/asset-limitations/)
- [Launch](https://www.contentstack.com/docs/developers/launch/about-launch)
- [Automation Hub Management API](https://www.contentstack.com/docs/developers/apis/automation-hub-management-api)

#### Algolia

- [Prepare records](https://www.algolia.com/doc/guides/sending-and-managing-data/prepare-your-data)
- [Algolia records](https://www.algolia.com/doc/guides/sending-and-managing-data/prepare-your-data/in-depth/what-is-in-a-record)
- [Structure ecommerce product records](https://www.algolia.com/doc/guides/sending-and-managing-data/prepare-your-data/how-to/ecommerce-records/)
- [Choosing between indices](https://www.algolia.com/doc/guides/sending-and-managing-data/prepare-your-data/in-depth/choosing-between-one-or-more-indices)
- [Send and update data](https://www.algolia.com/doc/guides/sending-and-managing-data/send-and-update-your-data/)
- [Manage indices](https://www.algolia.com/doc/guides/sending-and-managing-data/manage-indices-and-apps/manage-indices)
- [Replicas](https://www.algolia.com/doc/guides/managing-results/refine-results/sorting/in-depth/replicas)
- [Create replicas](https://www.algolia.com/doc/guides/managing-results/refine-results/sorting/how-to/creating-replicas)
- [Faceting](https://www.algolia.com/doc/guides/managing-results/refine-results/faceting)
- [Rules overview](https://www.algolia.com/doc/guides/managing-results/rules/rules-overview/)
- [Add banners](https://www.algolia.com/doc/guides/managing-results/rules/merchandising-and-promoting/how-to/add-banners)
- [Synonyms](https://www.algolia.com/doc/guides/managing-results/optimize-search-results/adding-synonyms/)
- [API keys](https://www.algolia.com/doc/guides/security/api-keys)
- [Secured API keys](https://www.algolia.com/doc/api-reference/api-methods/generate-secured-api-key/)
- [Click and conversion events](https://www.algolia.com/doc/guides/sending-events)
- [Service limits](https://www.algolia.com/doc/guides/scaling/algolia-service-limits/)
- [Record size limits](https://support.algolia.com/hc/en-us/articles/4406981897617-Is-there-a-size-limit-for-my-index-records)
- [Records and operations billing](https://support.algolia.com/hc/en-us/articles/17245378392977-How-does-Algolia-count-records-and-operations)
- [Rate limits](https://support.algolia.com/hc/en-us/articles/44485795695889-Rate-Limits)
- [commercetools product schema in Algolia](https://www.algolia.com/doc/integration/commercetools/indexing/product-schema)

#### Salesforce B2C Commerce / SFCC

- [OCAPI overview](https://developer.salesforce.com/docs/commerce/commerce-api/guide/get-started-with-ocapi.html)
- [OCAPI settings](https://developer.salesforce.com/docs/commerce/b2c-commerce/references/b2c-commerce-ocapi/ocapisettings.html)
- [OCAPI usage](https://developer.salesforce.com/docs/commerce/b2c-commerce/references/b2c-commerce-ocapi/apiusage.html)
- [OCAPI metadata](https://developer.salesforce.com/docs/commerce/b2c-commerce/references/b2c-commerce-ocapi/metadata.html)
- [Content Assets](https://developer.salesforce.com/docs/commerce/ocapi/guide/b2c-content-assets-for-developers.html)
- [Catalog Script API](https://developer.salesforce.com/docs/commerce/b2c-commerce/references/b2c-script-api/dw.catalog.Catalog.html)
- [Promotions for developers](https://developer.salesforce.com/docs/commerce/b2c-commerce/guide/b2c-promotions-for-developers.html)
- [Catalog object import/export](https://help.salesforce.com/s/articleView?id=cc.b2c_catalog_object_import_export.htm&type=5)

### 16.4 Questions vendor à poser

#### commercetools

- Limites actuelles du futur projet Etam via GraphQL `limits`: Product Types, variants, Standalone Prices, Product Selections, Stores, API Extensions, Subscriptions.
- Recommandation officielle Stores vs Projects pour FR + international Etam.
- Connecteur ou référence d'architecture CT -> Algolia recommandé.
- Stratégie prix promos complexes vs Product Discounts / Cart Discounts.

#### Contentstack

- Plan Etam: branches, workflows, publish rules, releases, asset quota, roles par langue/environnement, SSO/SCIM.
- Limites exactes applicables à la stack versionnée Etam: champs, entries, assets, API usage.
- Bonnes pratiques migration Page Designer SFCC vers Modular Blocks.
- Launch pertinent ou frontend hors Contentstack.

#### Algolia

- Plan Etam: records inclus, opérations, replicas standard/virtual, NeuralSearch, Recommend, Personalization, Analytics retention.
- Coût des records avec replicas selon structure SKU vs couleur.
- Reco officielle pour lingerie: record SKU + distinct ou record couleur.
- Gouvernance Merchandising Studio et rôles utilisateurs.

