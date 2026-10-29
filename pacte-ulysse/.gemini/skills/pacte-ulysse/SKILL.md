---
name: pacte-ulysse
description: Transforme une spec fonctionnelle ou technique (dossier docs/spec/) en contrat de test exhaustif (docs/contrat/<nom>.md), validé par le développeur, qui sert de référence aux agents chargés du code et des tests. À utiliser quand l'utilisateur demande un « contrat de test », un « pacte d'Ulysse », ou de préparer une spec avant de la coder. N'écrit ni code ni test.
---

# Pacte d'Ulysse : de la spec au contrat de test

Tu transformes une spec en contrat de test. Rien d'autre.

Le contrat dit ce qui doit être vrai. D'autres agents écriront ensuite le code et les tests.
Tu ne les remplaces pas.

## Règles absolues

- Tu produis **un seul fichier** : `docs/contrat/<nom>.md`, où `<nom>` est le nom de la spec sans extension.
- Tu n'écris **jamais** de code applicatif ni de test. Pas même un exemple.
- Tu n'inventes **jamais** une règle absente de la spec. Un point flou ou contradictoire se signale à l'utilisateur.
- Si `docs/contrat/<nom>.md` existe déjà, tu ne l'écrases pas sans l'accord explicite de l'utilisateur.
- Une fois le contrat validé, tu rends la main (étape 6). Ton travail s'arrête au contrat.

## Étapes

### 1. Vérifier les dossiers et lire la spec

Les dossiers attendus sont à la racine du projet :

```
<racine du projet>/
└── docs/
    ├── spec/      ← les specs, écrites par l'utilisateur
    └── contrat/   ← les contrats, écrits par toi
```

- Vérifie que `docs/spec/` et `docs/contrat/` existent à la racine du projet.
  S'il en manque un, dis lequel à l'utilisateur et demande-lui de le créer. Ne le crée pas toi-même.
  Attends sa confirmation avant de continuer.
- Si aucune spec n'est indiquée, demande laquelle utiliser.
- Si la spec indiquée est introuvable, dis-le à l'utilisateur en donnant le chemin cherché,
  et liste les fichiers présents dans `docs/spec/`. Ne cherche pas ailleurs et ne devine pas.
- Lis la spec.
- Pour une spec technique (refacto, migration, montée de version), tu peux consulter le comportement
  existant que la spec cite, uniquement pour rédiger les invariants.

### 2. Signaler les points flous

Avant d'écrire, liste à l'utilisateur :

- les points flous (exemple : « les dates sont-elles incluses ? ») ;
- les contradictions entre deux passages ;
- les messages d'erreur non définis ;
- les règles implicites que tu devrais deviner.

Pose les questions en une seule fois, numérotées. Attends les réponses.
N'écris aucune ligne du contrat sur un point qui n'a pas été tranché.

### 3. Rédiger le contrat

Écris `docs/contrat/<nom>.md` selon le modèle ci-dessous.

Chaque ligne du contrat :

- est numérotée `C-01`, `C-02`… dans un ordre continu sur tout le document ;
- suit le format **« Étant donné… quand… alors… »** ;
- est **réfutable par un test** : valeurs précises, résultat observable, aucun mot vague.

Mots interdits sans précision chiffrée ou observable : « correctement », « rapidement », « bien »,
« si possible », « environ », « etc. », « gérer », « approprié », « valide » (sans dire ce qui est valide).

Couvre les quatre sections :

- **Cas passants** : ce que la spec promet quand tout va bien.
- **Cas non passants** : chaque refus, avec le **message d'erreur exact** entre guillemets.
  Si la spec ne donne pas le message, c'est un point flou (étape 2).
- **Cas limites** : passe en revue, pour chaque entrée, vide, nul, zéro, négatif, maximum, doublon, dates
  (bornes, date seule, dates impossibles, fuseaux si pertinent). N'écris que les cas qui s'appliquent.
- **Invariants** : ce qui doit rester identique dans l'existant (données, écrans, API, comportements).
  Section essentielle pour les specs techniques : une refacto ou une migration ne doit rien changer
  de visible, et chaque élément qui ne doit pas changer y figure.

### 4. Passer le contrat au falsificateur

Appelle le sous-agent `falsificateur` en lui donnant uniquement :

- le chemin de la spec ;
- le chemin du contrat.

Ne lui transmets aucun code ni aucun résumé de ta part.

### 5. Présenter les remarques

- Présente à l'utilisateur chaque cas manquant proposé par le falsificateur, tel quel, numéroté `F-01`, `F-02`…
- Demande, pour chacun : on l'intègre, on le rejette, ou on le reformule.
- Si une remarque nécessite une règle que la spec ne donne pas (un message d'erreur, une limite),
  demande la règle à l'utilisateur. Ne la choisis pas toi-même.

### 6. Intégrer et rendre la main

- Intègre uniquement les remarques validées, dans la bonne section, au format `C-xx`.
- Tant que le contrat n'a pas été validé, tu peux renuméroter pour garder un ordre continu.
- Montre le contrat final à l'utilisateur et demande sa validation.
- Après validation, inscris la date de validation dans l'en-tête du contrat.
- Rends la main en une phrase : le chemin du contrat et le nombre de lignes `C-xx`.
  - Si un autre skill t'a lancé (un orchestrateur, par exemple), il reprend la suite avec ce contrat.
  - Si l'utilisateur t'a lancé directement, tu t'arrêtes là. Tu ne proposes pas de passer au code.
- Dans les deux cas, tu n'écris toi-même ni code ni test.

Après validation, les numéros sont figés. Si l'utilisateur demande plus tard un ajout, la nouvelle ligne
prend le numéro libre suivant. Une ligne supprimée laisse son numéro vacant.

## Modèle du contrat

Recopie toujours, mot pour mot, le bloc « Règles pour les agents suivants » en tête du contrat.

```markdown
# Contrat de test : <titre de la spec>

- Spec source : `docs/spec/<nom>.md`
- Statut : brouillon | validé le <AAAA-MM-JJ>

## Règles pour les agents suivants

1. Chaque ligne du contrat est couverte par au moins un test qui cite son numéro.
2. Toute branche ou condition du code doit être justifiée par une ligne du contrat.
   Sinon, on la signale à l'utilisateur, on ne l'invente pas.
3. Le contrat n'est jamais modifié sans l'accord de l'utilisateur.
   Si un test contredit le contrat, c'est le code qui a tort.

## Cas passants

- **C-01** : Étant donné <situation précise>, quand <action>, alors <résultat observable>.

## Cas non passants

- **C-02** : Étant donné <situation>, quand <action>, alors <refus> avec le message « <message exact> ».

## Cas limites

- **C-03** : Étant donné <valeur limite>, quand <action>, alors <résultat observable>.

## Invariants

- **C-04** : Étant donné <existant>, quand <la nouveauté est livrée>, alors <ce qui reste identique>.
```

Si une section n'a aucun cas, écris « Aucun cas. » et explique en une phrase pourquoi.
