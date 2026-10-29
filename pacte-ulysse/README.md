# pacte-ulysse

Un skill et un sous-agent pour Gemini CLI.
Ils transforment une spec en **contrat de test** validé par vous, avant toute ligne de code.
Le sous-agent **falsificateur** relit ce contrat pour y trouver les cas que personne n'a prévus
(voir [un cas réel : la panne d'Azure du 29 février 2012](#un-cas-réel--la-panne-mondiale-dazure-le-29-février-2012)).

```
pacte-ulysse/
├── README.md
└── .gemini/
    ├── skills/
    │   └── pacte-ulysse/
    │       └── SKILL.md        ← le skill
    └── agents/
        └── falsificateur.md    ← le sous-agent critique
```

---

## Le problème

Sur un projet peu testé, chaque livraison casse quelque chose. On corrige un bug, on en crée deux.

Avec un agent IA, c'est pire. Le même agent écrit le code **et** les tests.
Quand un test échoue, il a tendance à corriger le test plutôt que le code.
Le test passe au vert. Le bug reste.

Le problème n'est pas l'agent. C'est l'organisation : personne ne fixe la règle avant le jeu.

---

## La philosophie

Trois idées simples, empruntées à la psychologie et à la philosophie.

### 1. Le biais de confirmation

On cherche naturellement ce qui confirme ce qu'on pense déjà.

Celui qui écrit le code écrit des tests qui **confirment** son code.
Il ne teste pas ce à quoi il n'a pas pensé, puisqu'il n'y a pas pensé.

> Exemple. Un développeur code une remise de 10 % au-delà de 100 €.
> Il teste 150 € → 135 €. Le test passe.
> Il ne teste pas 100 € pile. Or la spec disait « à partir de 100 € ». Son code dit `> 100`.

**La solution** : séparer celui qui définit ce qui doit être vrai de celui qui le code.
Ici, le contrat est écrit avant le code, par un autre agent, et validé par vous.

### 2. La falsifiabilité (Karl Popper)

Pour Popper, une affirmation n'a de valeur scientifique que si on peut **prouver qu'elle est fausse**.
« Il pleuvra un jour » n'apprend rien. « Il pleuvra demain à Lyon » peut être vérifié.

Appliqué aux tests :

- « L'export fonctionne correctement » : aucun test ne peut le réfuter. Inutile.
- « Étant donné 3 factures en mars, quand j'exporte mars, alors le fichier contient 3 lignes » :
  un test peut le faire échouer. Utile.

Chaque ligne du contrat doit donc pouvoir être prise en défaut.

Et un deuxième agent, le **falsificateur**, relit le contrat dans cet esprit.
Il ne cherche pas à le confirmer. Il cherche ce qui pourrait échouer sans que personne ne le voie.

### 3. Le pacte d'Ulysse (le précommitment)

Ulysse veut entendre le chant des sirènes sans y céder.
Il se fait attacher au mât **avant** d'arriver près d'elles, et ordonne à l'équipage de ne pas le détacher,
même s'il le supplie.

Il décide quand il a la tête froide. Il s'interdit de changer d'avis sous la tentation.

Ici, la tentation, c'est le test rouge qu'on « ajuste » pour qu'il passe.
Le contrat est donc validé **avant** tout code. Ensuite, il ne bouge plus sans votre accord.

**Si un test contredit le contrat, c'est le code qui a tort.**

---

## Le falsificateur

Le falsificateur est un sous-agent qui relit le contrat avec un seul but : trouver ce qui manque.

- Il lit la spec et le contrat. **Jamais le code**, pour ne pas être influencé par ce qui existe déjà.
- Il ne modifie rien. Il rend une liste de cas manquants, numérotés `F-01`, `F-02`…
- Il n'invente aucune règle. Quand la spec ne dit rien, il écrit « règle à définir par l'utilisateur ».
  C'est vous qui tranchez.

Il cherche systématiquement :

- les entrées invalides : champ absent, mauvais format, caractères spéciaux ;
- les droits : qui n'a pas le droit de faire l'action ?
- les messages d'erreur jamais définis ;
- les valeurs limites : vide, zéro, négatif, maximum, doublon, bornes de dates, date impossible ;
- les enchaînements incohérents : action répétée, dans le mauvais ordre, sur un élément supprimé ;
- les effets de bord sur l'existant.

Il ne trouve pas de bug dans le code. Il trouve les **questions que personne n'a posées**.
Et une question posée avant le code coûte une minute. Après la mise en production, elle peut coûter une panne.

### Un cas réel : la panne mondiale d'Azure, le 29 février 2012

Ce bug est documenté publiquement par Microsoft
([rapport d'incident](https://azure.microsoft.com/en-us/blog/summary-of-windows-azure-service-disruption-on-feb-29th-2012/)).

**La règle.** Chaque serveur reçoit un certificat de sécurité, valable un an à partir de sa création.

Une spec qui le dirait ressemblerait à ceci :

> En tant que plateforme, je veux créer un certificat valable 1 an,
> afin de sécuriser les échanges entre serveurs.

La phrase paraît claire. Personne n'aurait posé de question.

**Le code.** Les développeurs ont écrit « date du jour, année + 1 ». Pendant des années, ça a marché.

Le 29 février 2012, le code a calculé une expiration au 29 février 2013.
Cette date n'existe pas. La création des certificats a échoué.
Les nouveaux serveurs ne pouvaient plus démarrer.
Azure a été en panne pendant de nombreuses heures, dans le monde entier.

**Ce que le falsificateur aurait fait.** Sa consigne, dans `falsificateur.md`, lui demande de chercher
les « valeurs limites : … bornes de dates, date seule, date impossible ». Sur cette spec, il aurait rendu :

```markdown
- **F-01** [Cas limites] : Étant donné un certificat créé le 29/02/2012, quand on calcule sa date d'expiration, alors <règle à définir par l'utilisateur>.
```

Le développeur aurait répondu « le 28/02/2013 » ou « le 01/03/2013 ».
La réponse devenait une ligne du contrat, donc un test. Le bug n'aurait jamais existé.

---

## Ce que fait le skill

- Il lit une spec, fonctionnelle (« l'utilisateur peut exporter… ») ou technique
  (refacto, migration, montée de version).
- Il vous signale les points flous ou contradictoires, au lieu de deviner.
- Il écrit un contrat de test : `docs/contrat/<nom>.md`, avec des lignes numérotées `C-01`, `C-02`…
  au format « Étant donné… quand… alors… ».
- Il couvre quatre sections : cas passants, cas non passants (avec le message d'erreur exact),
  cas limites, invariants.
- Il envoie le contrat au falsificateur, vous présente ses remarques, intègre celles que vous validez.
- Il rend la main : à vous, ou au skill qui l'a lancé (un orchestrateur, par exemple).

## Ce qu'il ne fait pas

- Il n'écrit pas de code.
- Il n'écrit pas de tests.
- Il ne choisit pas à votre place une règle absente de la spec.
- Il ne modifie pas un contrat existant sans votre accord.

Le code et les tests sont le travail d'autres agents, qui reçoivent le contrat comme référence.

---

## Installation

Prérequis : Gemini CLI avec les Agent Skills et les sous-agents (activés par défaut dans les versions récentes).

### Pour un projet

Copiez le dossier `.gemini/` à la racine de votre projet :

```bash
cp -r pacte-ulysse/.gemini/ /chemin/vers/mon-projet/
```

Si votre projet a déjà un dossier `.gemini/`, copiez seulement les deux fichiers :

```bash
mkdir -p mon-projet/.gemini/skills/pacte-ulysse mon-projet/.gemini/agents
cp pacte-ulysse/.gemini/skills/pacte-ulysse/SKILL.md mon-projet/.gemini/skills/pacte-ulysse/
cp pacte-ulysse/.gemini/agents/falsificateur.md      mon-projet/.gemini/agents/
```

### Pour tous vos projets

Copiez les mêmes fichiers dans votre dossier utilisateur :

```bash
mkdir -p ~/.gemini/skills/pacte-ulysse ~/.gemini/agents
cp pacte-ulysse/.gemini/skills/pacte-ulysse/SKILL.md ~/.gemini/skills/pacte-ulysse/
cp pacte-ulysse/.gemini/agents/falsificateur.md      ~/.gemini/agents/
```

### Vérifier

Dans Gemini CLI :

```
/skills list
/agents
```

`pacte-ulysse` doit apparaître dans la liste des skills, `falsificateur` dans celle des agents.

Si les sous-agents ont été désactivés, vérifiez que `settings.json` ne contient pas
`"experimental": { "enableAgents": false }`.

---

## Utilisation

### 1. Créer les dossiers et déposer la spec

À la racine du projet, créez un dossier `docs/` avec deux sous-dossiers, `spec/` et `contrat/` :

```bash
mkdir -p docs/spec docs/contrat
```

Placez ensuite la spec dans `docs/spec/` :

```
mon-projet/
└── docs/
    ├── spec/
    │   └── export-factures.md     ← écrite par vous
    └── contrat/
        └── export-factures.md     ← écrit par le skill
```

Le skill ne crée pas ces dossiers. S'il en manque un, il vous demande de le créer.
Si la spec indiquée est introuvable, il vous le dit et liste les specs présentes dans `docs/spec/`.

Une spec peut être courte. Elle doit dire ce qui est attendu, pas comment le coder.

### 2. Appeler le skill

```
Utilise pacte-ulysse sur docs/spec/export-factures.md
```

Gemini CLI vous demande l'autorisation d'activer le skill. Acceptez.

### 3. Les étapes de validation

| Étape | Qui agit | Ce que vous faites |
|---|---|---|
| 1. Questions | Le skill liste les points flous de la spec | Vous répondez |
| 2. Brouillon | Le skill écrit `docs/contrat/export-factures.md` | Rien |
| 3. Critique | Le falsificateur propose des cas manquants `F-01`, `F-02`… | Rien |
| 4. Tri | Le skill vous présente chaque cas manquant | Vous dites : intégrer, rejeter ou reformuler |
| 5. Validation | Le skill montre le contrat final | Vous validez |

Après la validation, le skill note la date dans le contrat et rend la main.
S'il a été lancé par un autre skill, celui-ci reprend la suite avec le contrat. Sinon, il s'arrête là.

Prenez le temps de l'étape 4. C'est là que vous vous attachez au mât.

---

## Exemple complet

### La spec : `docs/spec/export-factures.md`

```markdown
# Export des factures

Le comptable peut exporter les factures d'une période au format CSV.
Il choisit une date de début et une date de fin.
Le fichier contient, pour chaque facture : numéro, date, client, montant TTC.
Les factures annulées ne sont pas exportées.
Si aucune facture ne correspond, afficher « Aucune facture sur cette période. »
```

### Étape 1 : les questions du skill

Le skill ne devine pas. Il demande :

> 1. Les dates de début et de fin sont-elles incluses ?
> 2. Quel message afficher si la date de début est après la date de fin ?
> 3. Dans quel ordre trier les lignes ?
> 4. Quel séparateur et quel format pour les dates et les montants ?

Réponses du développeur :

> 1. Oui, incluses.
> 2. « La date de début doit précéder la date de fin. »
> 3. Par date croissante, puis par numéro.
> 4. Séparateur `;`, dates `JJ/MM/AAAA`, montants avec virgule et deux décimales, sans symbole €.

### Étape 3 : les remarques du falsificateur

Le falsificateur lit la spec et le brouillon, jamais le code. Il rend :

```markdown
- **F-01** [Cas non passants] : Étant donné un utilisateur qui n'est pas comptable, quand il demande un export, alors <règle à définir par l'utilisateur>.
- **F-02** [Cas non passants] : Étant donné une date de début ou de fin absente, quand on lance l'export, alors <règle à définir par l'utilisateur>.
- **F-03** [Cas non passants] : Étant donné une date impossible (31/02/2026), quand on lance l'export, alors <règle à définir par l'utilisateur>.
- **F-04** [Cas limites] : Étant donné une période qui ne contient que des factures annulées, quand on exporte, alors le message « Aucune facture sur cette période. » s'affiche.
- **F-05** [Cas limites] : Étant donné un nom de client qui contient un « ; », quand on exporte, alors <règle à définir par l'utilisateur>.
- **F-06** [Cas limites] : Étant donné un avoir de montant négatif, quand on exporte, alors <règle à définir par l'utilisateur>.
- **F-07** [Cas limites] : Étant donné deux factures avec le même numéro, quand on exporte, alors <règle à définir par l'utilisateur>.
- **F-08** [Invariants] : Étant donné une facture exportée, quand l'export est terminé, alors la facture n'est pas modifiée en base.
```

Tri du développeur :

- F-01 : intégrer, message « Accès réservé aux comptables. »
- F-02 : intégrer, message « Les dates de début et de fin sont obligatoires. »
- F-03 : intégrer, message « Date invalide. »
- F-04 : intégrer.
- F-05 : intégrer, le champ est entouré de guillemets.
- F-06 : intégrer, le montant est exporté avec son signe moins.
- F-07 : **rejeter**. Le numéro de facture est unique en base, ce cas ne peut pas arriver.
- F-08 : intégrer.

### Le contrat obtenu : `docs/contrat/export-factures.md`

```markdown
# Contrat de test : Export des factures

- Spec source : `docs/spec/export-factures.md`
- Statut : validé le 2026-10-24

## Règles pour les agents suivants

1. Chaque ligne du contrat est couverte par au moins un test qui cite son numéro.
2. Toute branche ou condition du code doit être justifiée par une ligne du contrat.
   Sinon, on la signale à l'utilisateur, on ne l'invente pas.
3. Le contrat n'est jamais modifié sans l'accord de l'utilisateur.
   Si un test contredit le contrat, c'est le code qui a tort.

## Cas passants

- **C-01** : Étant donné un comptable et 3 factures émises entre le 01/03/2026 et le 31/03/2026, quand il exporte du 01/03/2026 au 31/03/2026, alors il reçoit un fichier CSV de 4 lignes : 1 ligne d'en-tête et 3 lignes de factures.
- **C-02** : Étant donné un export réussi, quand on lit la première ligne du fichier, alors elle vaut exactement `numero;date;client;montant_ttc`.
- **C-03** : Étant donné la facture F-2026-042 du 15/03/2026, client « Dupont SARL », 1 234,50 € TTC, quand elle est exportée, alors sa ligne vaut exactement `F-2026-042;15/03/2026;Dupont SARL;1234,50`.
- **C-04** : Étant donné une facture annulée datée dans la période, quand le comptable exporte, alors cette facture n'apparaît pas dans le fichier.
- **C-05** : Étant donné les factures F-2 du 10/03, F-1 du 10/03 et F-3 du 05/03, quand le comptable exporte mars, alors les lignes sont dans l'ordre F-3, F-1, F-2.

## Cas non passants

- **C-06** : Étant donné un utilisateur qui n'est pas comptable, quand il demande un export, alors l'export est refusé avec le message « Accès réservé aux comptables. » et aucun fichier n'est produit.
- **C-07** : Étant donné une date de début au 31/03/2026 et une date de fin au 01/03/2026, quand le comptable lance l'export, alors il est refusé avec le message « La date de début doit précéder la date de fin. » et aucun fichier n'est produit.
- **C-08** : Étant donné une date de début ou une date de fin absente, quand le comptable lance l'export, alors il est refusé avec le message « Les dates de début et de fin sont obligatoires. » et aucun fichier n'est produit.
- **C-09** : Étant donné la date de début 31/02/2026, quand le comptable lance l'export, alors il est refusé avec le message « Date invalide. » et aucun fichier n'est produit.

## Cas limites

- **C-10** : Étant donné aucune facture entre le 01/03/2026 et le 31/03/2026, quand le comptable exporte cette période, alors le message « Aucune facture sur cette période. » s'affiche et aucun fichier n'est produit.
- **C-11** : Étant donné une période qui ne contient que des factures annulées, quand le comptable l'exporte, alors le message « Aucune facture sur cette période. » s'affiche et aucun fichier n'est produit.
- **C-12** : Étant donné une facture du 01/03/2026 et une facture du 31/03/2026, quand le comptable exporte du 01/03/2026 au 31/03/2026, alors les deux factures sont dans le fichier.
- **C-13** : Étant donné une date de début et une date de fin identiques (15/03/2026), quand le comptable exporte, alors le fichier contient uniquement les factures du 15/03/2026.
- **C-14** : Étant donné une facture de 0,00 € TTC, quand elle est exportée, alors son montant vaut `0,00`.
- **C-15** : Étant donné un avoir de -50,00 € TTC, quand il est exporté, alors son montant vaut `-50,00`.
- **C-16** : Étant donné le client « Martin; Fils », quand sa facture est exportée, alors le champ client vaut exactement `"Martin; Fils"`.

## Invariants

- **C-17** : Étant donné une facture exportée, quand l'export est terminé, alors son statut, son montant et sa date de dernière modification en base sont identiques à ceux d'avant l'export.
```

Remarquez :

- Aucune ligne ne dit « correctement ». Chacune peut échouer à un test précis.
- Le cas F-07 n'apparaît pas : le développeur l'a rejeté, et ce choix lui appartient.
- Les messages d'erreur viennent du développeur, pas de l'agent.

---

## Transmettre le contrat aux agents de développement

Le contrat est un simple fichier Markdown. Tout agent peut le lire.

### Le principe

- L'agent qui écrit le code et celui qui écrit les tests reçoivent **le contrat**, pas seulement la spec.
- Ils commencent par lire les « Règles pour les agents suivants », en tête du contrat.
- Chaque test cite le numéro de la ligne qu'il couvre, dans son nom ou en commentaire.
  Exemple : `test_C07_date_debut_apres_date_fin`.

### Phrases types

Pour l'agent qui écrit les tests :

```
Écris les tests unitaires de docs/contrat/export-factures.md.
Respecte les règles en tête du contrat. Chaque test cite son numéro C-xx.
N'écris pas le code de production.
```

Pour l'agent qui écrit le code :

```
Implémente docs/spec/export-factures.md.
Le contrat docs/contrat/export-factures.md fait foi : lis ses règles en tête.
Ne modifie ni le contrat ni les tests. Si un test te semble faux, signale-le-moi.
```

Si possible, utilisez une session distincte pour chaque agent.
Celui qui code ne doit pas être celui qui a écrit les tests.

### Vérifier la couverture

Chaque numéro du contrat doit apparaître dans au moins un test :

```bash
grep -o "C-[0-9]\+" docs/contrat/export-factures.md | sort -u > attendus.txt
grep -rhoE "C-?[0-9]+" tests/ | sed -E 's/C-?/C-/' | sort -u > couverts.txt
comm -23 attendus.txt couverts.txt   # numéros sans test
```

### Quand un test échoue

Rappelez la règle à l'agent : **c'est le code qui a tort**.

Si vous pensez que c'est le contrat qui a tort, c'est à vous de le modifier, en connaissance de cause.
Pas à l'agent.
