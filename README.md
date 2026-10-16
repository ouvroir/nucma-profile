# Profil nucma pour les TEI Stylesheets

Profil `nucma` des [TEI Stylesheets](https://github.com/teic/stylesheets) : conversion en TEI des entrées de l’Encyclopédie NUCMA rédigées dans Word, avec en-tête rempli à partir du document, nettoyage des artefacts de Word et sortie indentée.

La normalisation typographique et le nettoyage des fichiers déjà importés se trouvent dans le dépôt [typesetter](https://codeberg.org/Factoton/typesetter) ; les textes et `meta/authors.xml` dans `nucma-textes`.

| Fichier | Rôle |
|---|---|
| `docx/from.xsl` | Conversion d’un `.docx` en TEI (surcharge de `docxtotei`). |
| `docx/to.xsl`, `docx/template.docx`, `docx/headersfooters.xml` | Fichiers du profil pour la conversion TEI → Word. |

`from.xsl` est documenté en détail dans son en-tête (`xd:doc`, format de documentation d’oXygen).

## Installation

- **TEI Stylesheets** installées avec Ant et disponibles dans le PATH.
- Déclarer le profil par un lien symbolique dans les Stylesheets :

  ```sh
  ln -s /path/to/nucma-profile /path/to/TEIC/Stylesheets/profiles/nucma
  ```

  Le profil doit être appelé par ce lien : `from.xsl` importe `../../../docx/from/docxtotei.xsl`, chemin relatif aux Stylesheets.

- Le profil lit `meta/authors.xml` du dépôt des textes au chemin indiqué par le paramètre `nucma-authors` de `from.xsl` (chemin sur le poste d’Emmanuel Château-Dutier). Sur un autre poste, modifier ce chemin ou le donner par une variable d’environnement :

  ```sh
  NUCMA_AUTHORS=file:///chemin/vers/nucma-textes/meta/authors.xml docxtotei --profile=nucma Entree.docx entree.xml
  ```

## Importer un texte

```sh
docxtotei --profile=nucma Variabilite.docx variabilite.xml
```

Le TEI produit est déjà nettoyé et indenté, et son en-tête est rempli (titre, auteurs, identifiant). Variables d’environnement :

| Variable | Défaut | Effet |
|---|---|---|
| `NUCMA_CHAPTER` | `acquisition` | chapitre de l’identifiant et de `seriesStmt` |
| `NUCMA_SCHEMAS` | `../../schemas/` | chemin vers `schemas/` dans les instructions `xml-model` ; le défaut convient à un fichier rangé dans `fr/<chapitre>/` ou `en/<chapitre>/` |
| `NUCMA_RESP` | `ECD` | personne responsable de la conversion dans `revisionDesc` |
| `NUCMA_AUTHORS` | chemin local | URI de `meta/authors.xml` |

Le fichier produit est associé au schéma (`nucma.rng`) et au Schematron (`nucma-quality.sch`) par deux instructions `xml-model` en tête de fichier ; oXygen les applique dès l’ouverture. La conversion est notée dans `revisionDesc` (`<change when="…" who="#ECD">Conversion depuis Word (profil nucma)</change>`).

Viennent ensuite la structuration manuelle dans oXygen, puis la normalisation typographique (voir [typesetter](https://codeberg.org/Factoton/typesetter)).

## En-tête

Le profil lit le bloc d’en-tête écrit au début du document Word, en français ou en anglais, le retire du texte et remplit le `teiHeader` :

```text
Titre de l’entrée :                        Entry Title:
Négocier la variabilité                    Collections in Shared Ownership
Titre court : Variabilité                  Short Title: Co-acquisition
Auteur·rices :                             Authors:
• Anne Bénichou, professeure, UQAM         • Jessica Minier, PhD Candidate, UQO
```

- **Titre** : un seul `<title>` sans `@type` ; s’il est suivi d’un sous-titre, `<title type="main">` et `<title type="sub">`.
- **Titre court** : il sert de nom de fichier et d’identifiant de l’entrée, sans majuscule ni diacritique, sur `sourceDesc` : `<sourceDesc xml:id="nucma.fr.acquisition.variabilite">`. La langue vient des étiquettes (`fr` ou `en`), le chapitre de `NUCMA_CHAPTER`.
- **Auteurs** : chaque ligne « Nom, rôle, affiliation » devient un `<author>` avec `persName`, `roleName` et `affiliation`. L’auteur est cherché dans `meta/authors.xml` : s’il y figure, il reçoit son `@ref` (`#benichouAnne`), son prénom et son nom tels qu’ils y sont découpés, et ses identifiants (ORCID, ISNI…). Sinon, le dernier mot du nom est pris comme nom de famille et un `<!-- @todo -->` demande de vérifier.
- **Contributeurs** : les personnes listées sous un sous-titre du bloc des auteurs (« Research Assistant: », « Assistante de recherche : ») ne sont pas des auteurs : elles deviennent des `<respStmt>` (`<resp>Research Assistant</resp>`, `persName` avec `roleName`, `orgName` pour l’affiliation), cherchées elles aussi dans `meta/authors.xml` (liste « Autres contributeurs »).
- Les puces tapées à la main (« ● Julie Bawin, … ») et les espaces insécables dans les étiquettes (« Titre⍽de l’entrée⍽: ») sont acceptées.
- **Reste de l’en-tête** : complété d’après le modèle des entrées du corpus, en français ou en anglais : `publicationStmt` (éditeur, année, licence CC BY 4.0, DOI à compléter), `seriesStmt` (« Chapitre Acquisition », en anglais « Chapter: Acquisition », titre de l’encyclopédie, URI), `sourceDesc` (« Fichier nativement numérique ») et `profileDesc/langUsage`. L’`editionStmt` vide produit par la conversion est supprimé.
- **Langue** : posée sur `<text xml:lang="fr">` (ou `en`) d’après les étiquettes ; la normalisation applique ensuite la typographie de cette langue, quel que soit le dossier du fichier.

## Nettoyage

Le profil applique les règles de nettoyage de `import-cleanup.xsl` (dépôt [typesetter](https://codeberg.org/Factoton/typesetter), où elles sont décrites et testées) : styles de police, `hi` et `seg` vides, `@xml:space`, commentaires de Word en `<!-- @todo … -->`, notes regroupées dans `back/div[@type='notes']` (`note01`, `note02`…), ancres et intertitres vides, `@type="external"` sur les liens hors de `nucma.ca`, attributs de `graphic`, espaces insécables orphelines.

## Maintenance

Les règles de nettoyage existent en deux exemplaires, dans `docx/from.xsl` et dans `import-cleanup.xsl` du dépôt typesetter. Toute modification doit être faite dans les deux, et les tests XSpec de typesetter (`test/import-cleanup.xspec`) relancés.
