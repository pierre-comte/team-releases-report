# Spécification fonctionnelle & technique — Release Quality Control

## 1. Objet

Mettre en place un outil Python de **Release Quality Control** permettant de contrôler automatiquement la conformité du processus de livraison de trois équipes IT bancaires : **MAKS, GATEWAY et PKEM**.

L'outil doit rapprocher les informations issues de **Jira**, de **Git / Pull Requests** et des **release notes Git**, sans intégration directe Jira ↔ Git et sans webhook.

Le résultat principal attendu est un **dashboard HTML lisible par un manager**, complété par des détails par équipe, release et Jira, ainsi qu'une liste d'anomalies actionnables.

---

## 2. Contexte et contraintes

### 2.1 Outils

- Jira : source de vérité du processus fonctionnel et du workflow.
- Git : source de vérité technique pour les Pull Requests et les release notes.
- Les revues de code sont réalisées via PR.
- Les équipes ne disposent pas / ne sont pas autorisées à utiliser de Hook Jira ↔ Git.
- Le développement de l'outil se fera en **Python**.

### 2.2 Processus Jira cible

Cycle de vie attendu :

```text
NEW
  |
  | BA ou TL analyse le besoin
  | label "Analyzed"
  v
ANALYZED
  |
  | TL contrôle
  | - estimation temps
  | - golden rules
  | - services impacted
  | - transition Ready to Dev
  v
READY TO DEV
  |
  | affectation développeur
  v
IN PROGRESS
  |
  | développeur renseigne la PR
  v
CODE REVIEW
  |
  v
...
```

### 2.3 Règles de processus

1. Un Jira est créé en statut `New`.
2. Le BA ou un TL analyse le Jira et ajoute le label `Analyzed`.
3. Un TL contrôle le Jira.
4. Le TL renseigne :
   - estimation du temps ;
   - golden rules ;
   - services impacted.
5. Le TL fait passer le Jira à `Ready to Dev`.
6. Le Jira est affecté à un développeur et passe à `In Progress`.
7. Lors du passage à `Code Review`, le développeur doit renseigner la PR associée.
8. Une release Jira peut contenir plusieurs Jira.
9. Les release notes sont stockées dans Git selon une convention de dossiers par release et date de MEP.

---

# 3. Objectifs fonctionnels

## 3.1 Objectif principal

Pour chaque release Jira, déterminer si chaque Jira respecte les quality gates suivants :

- analyse effectuée ;
- validation effectuée par un TL ;
- champs obligatoires renseignés ;
- PR associée ;
- PR existante dans Git ;
- PR éventuellement mergée ;
- release note présente ;
- cohérence entre Jira et PR ;
- cohérence entre Jira, release et date de MEP.

## 3.2 Objectifs secondaires

Produire :

- un reporting global mensuel ;
- un reporting par équipe ;
- un reporting par release ;
- une liste d'anomalies ;
- des KPI de qualité ;
- un détail audit-able pour chaque Jira ;
- un historique des rapports.

---

# 4. Périmètre du POC

Le POC doit volontairement rester limité.

### Périmètre initial

- 1 équipe, idéalement MAKS ;
- 1 release ;
- environ 10 à 20 Jira ;
- 4 quality gates principales :
  1. `Analyzed`;
  2. validation TL ;
  3. PR ;
  4. release note.

### Périmètre cible

- équipes MAKS, GATEWAY, PKEM ;
- toutes les releases d'un mois ;
- reporting mensuel ;
- historique des rapports ;
- automatisation périodique.

---

# 5. Principe général d'architecture

Architecture en trois couches :

```text
             +------------------+
             |      Jira        |
             +--------+---------+
                      |
                      v
              +---------------+
              |  Collecteurs  |
              +-------+-------+
                      |
             +--------+--------+
             |                 |
             v                 v
       +-----------+     +-----------+
       | Jira data |     |  Git data |
       +-----------+     +-----------+
             |                 |
             +--------+--------+
                      |
                      v
              +---------------+
              | Quality Engine|
              +-------+-------+
                      |
                      v
              +---------------+
              | Quality Report|
              +-------+-------+
                      |
                      v
              +---------------+
              | HTML Reporter |
              +---------------+
```

Principe :

> Les connecteurs Jira et Git collectent les données. Le moteur de contrôle applique les règles métier. Le reporter présente les résultats.

Les règles métier ne doivent pas être mélangées avec les appels API.

---

# 6. Architecture technique Python

Structure recommandée :

```text
quality-release-report/
│
├── config/
│   └── config.yml
│
├── src/
│   ├── jira_client.py
│   ├── git_client.py
│   ├── release_notes.py
│   ├── checks.py
│   ├── models.py
│   ├── collector.py
│   └── reporter.py
│
├── templates/
│   └── report.html.j2
│
├── tests/
│   ├── test_jira.py
│   ├── test_git.py
│   ├── test_checks.py
│   └── fixtures/
│       ├── jira_release.json
│       ├── jira_issue.json
│       ├── jira_changelog.json
│       ├── git_pr.json
│       └── release_notes.json
│
├── output/
│   └── report.html
│
├── main.py
├── requirements.txt
└── README.md
```

---

# 7. Modèle de configuration

La configuration métier doit être externalisée.

Exemple :

```yaml
jira:
  project_keys:
    - MAKS
    - GATEWAY
    - PKEM

workflow:
  analyzed_label: "Analyzed"
  ready_to_dev_status: "Ready to Dev"
  in_progress_status: "In Progress"
  code_review_status: "Code Review"

teams:
  MAKS:
    jira_projects:
      - MAKS
    tech_leads:
      - john.doe
      - jane.doe

  GATEWAY:
    jira_projects:
      - GATEWAY
    tech_leads:
      - bob.smith

  PKEM:
    jira_projects:
      - PKEM
    tech_leads:
      - alice.smith

git:
  repository: "repository-name"

release_notes:
  root_path: "release-notes"
```

Les utilisateurs TL et les paramètres spécifiques aux équipes ne doivent pas être codés en dur.

---

# 8. Connecteur Jira

Créer une classe dédiée :

```python
class JiraClient:

    def get_releases(self, start_date, end_date):
        ...

    def get_release_issues(self, release):
        ...

    def get_issue(self, issue_key):
        ...

    def get_issue_history(self, issue_key):
        ...
```

Responsabilité :

- authentification ;
- appels API ;
- pagination ;
- récupération des données ;
- gestion des erreurs ;
- mapping vers les modèles internes.

Le connecteur ne doit pas décider si un Jira est conforme.

---

# 9. Connecteur Git

Créer une abstraction :

```python
class GitClient:

    def get_pull_request(self, url):
        ...

    def search_pull_request(self, issue_key):
        ...

    def get_pull_request_details(self, pr):
        ...
```

Le connecteur devra être adapté à la forge utilisée : GitLab, GitHub, Bitbucket, Azure DevOps, etc.

Informations minimales attendues sur une PR :

- URL ;
- titre ;
- auteur ;
- repository ;
- état ;
- date de création ;
- date de merge ;
- reviewers ;
- branche source ;
- branche cible.

---

# 10. Association Jira ↔ PR

## 10.1 Méthode recommandée

Utiliser un champ Jira dédié contenant l'URL de la PR.

Exemple :

```text
Pull Request:
https://git.company.com/team/project/pull/123
```

Si plusieurs PR sont possibles, utiliser un champ multi-valeurs ou multi-lignes.

## 10.2 Vérification complémentaire

Même si l'URL est renseignée dans Jira, contrôler que la clé Jira apparaît dans la PR.

Exemple :

```text
MAKS-1234 - Ajout du traitement X
```

Contrôles :

```text
Jira -> URL PR -> PR existe
Jira -> clé Jira présente dans PR
```

Cela permet de détecter une mauvaise association.

---

# 11. Analyse de l'historique Jira

L'historique Jira est indispensable.

Le système ne doit pas uniquement regarder l'état actuel du Jira.

Il doit rechercher les transitions historiques.

Contrôle principal :

```text
New -> Ready to Dev
```

Informations à extraire :

- auteur de la transition ;
- date ;
- statut précédent ;
- statut suivant.

Le système doit comparer l'auteur à la liste des TL configurés.

Exemple :

```text
New -> Ready to Dev
Author: john.doe
john.doe est TL de MAKS
=> PASS
```

Contre-exemple :

```text
New -> Ready to Dev
Author: developer01
developer01 n'est pas TL
=> FAIL
```

---

# 12. Contrôle du label Analyzed

Le système doit vérifier que le label :

```text
Analyzed
```

a été ajouté avant le passage à `Ready to Dev`.

Ne pas se limiter à :

```text
label présent actuellement
```

mais rechercher l'événement historique si l'API Jira permet de l'identifier.

Règle :

```text
Analyzed avant Ready to Dev = PASS
Analyzed absent = FAIL
Analyzed après Ready to Dev = FAIL
```

---

# 13. Contrôle des champs obligatoires

Avant ou lors du passage à `Ready to Dev`, contrôler la présence de :

- estimation temps ;
- golden rules ;
- services impacted.

Les noms techniques des champs doivent être configurables.

Exemple :

```yaml
required_fields:
  estimation: "customfield_xxxxx"
  golden_rules: "customfield_xxxxx"
  services_impacted: "customfield_xxxxx"
```

Résultat possible :

```text
Estimation          PASS
Golden Rules        PASS
Services impacted   FAIL
```

---

# 14. Contrôle de la PR

À partir du champ Jira :

```text
Pull Request URL
```

contrôler :

1. URL présente ;
2. PR accessible ;
3. PR correspondante au Jira ;
4. PR non abandonnée ;
5. PR mergée si la règle métier l'exige.

Pour le POC, le minimum est :

```text
PR renseignée
PR existante
```

Le contrôle `merged` peut être ajouté immédiatement si l'API Git le permet.

---

# 15. Contrôle des release notes

Convention cible :

```text
release-notes/
└── <release>/
    └── <date-MEP>/
        ├── MAKS-123.md
        ├── MAKS-124.md
        └── MAKS-125.md
```

Le système doit rechercher une release note correspondant à chaque Jira.

Règle :

```text
Jira dans release
+
release note correspondante
=
PASS
```

Sinon :

```text
RELEASE_NOTE_MISSING
```

La convention exacte d'arborescence devra être paramétrable.

---

# 16. Quality checks

Créer des règles indépendantes.

Exemples :

```python
check_analyzed(issue)
check_tech_lead_validation(issue)
check_required_fields(issue)
check_pr_exists(issue)
check_pr_is_valid(issue)
check_release_note(issue)
```

Chaque check retourne un résultat structuré.

Exemple :

```python
@dataclass
class CheckResult:
    code: str
    status: str
    message: str
```

Valeurs de `status` :

- `PASS`
- `WARNING`
- `FAIL`

---

# 17. Codes d'anomalies

Liste initiale :

```text
ANALYZED_LABEL_MISSING
ANALYZED_LABEL_LATE

TL_VALIDATION_MISSING
TL_VALIDATION_INVALID

REQUIRED_FIELD_MISSING

PR_MISSING
PR_NOT_FOUND
PR_NOT_MATCHING_JIRA
PR_NOT_MERGED

RELEASE_NOTE_MISSING
RELEASE_NOTE_NOT_FOUND
```

Extensions possibles :

```text
PR_AFTER_RELEASE
JIRA_ADDED_TO_RELEASE_LATE
MULTIPLE_PR
PR_WRONG_TARGET_BRANCH
```

Les codes doivent être stables afin de permettre des statistiques dans le temps.

---

# 18. Modèles de données

Exemple :

```python
@dataclass
class PullRequest:
    url: str
    title: str
    author: str
    state: str
    merged_at: Optional[datetime]


@dataclass
class JiraIssue:
    key: str
    summary: str
    status: str
    assignee: Optional[str]
    labels: list[str]
    fix_versions: list[str]
    pr_urls: list[str]


@dataclass
class CheckResult:
    code: str
    status: str
    message: str


@dataclass
class JiraQualityReport:
    issue: JiraIssue
    checks: list[CheckResult]
```

Le modèle pourra évoluer selon les informations réellement disponibles via les API.

---

# 19. QualityReport

Créer un objet agrégateur :

```python
@dataclass
class QualityReport:
    release_name: str
    team: str
    issues: list[JiraQualityReport]

    @property
    def total(self):
        return len(self.issues)
```

Il doit permettre de calculer :

- nombre total de Jira ;
- nombre conformes ;
- nombre en warning ;
- nombre en erreur ;
- taux de couverture PR ;
- taux de validation TL ;
- taux de release notes ;
- nombre d'anomalies par code.

---

# 20. KPI du dashboard

## KPI principaux

### PR Coverage

```text
Jira avec PR / nombre total de Jira
```

### TL Validation

```text
Jira validés par un TL / nombre total de Jira
```

### Release Notes

```text
Jira avec release note / nombre total de Jira
```

### Process Compliance

```text
Jira sans aucune anomalie / nombre total de Jira
```

Exemple :

```text
100 Jira
89 conformes

Process Compliance = 89 %
```

---

# 21. Score de qualité

Un score peut être ajouté mais ne doit jamais masquer les anomalies critiques.

Proposition initiale :

| Contrôle | Poids |
|---|---:|
| PR associée | 40 % |
| PR mergée | 20 % |
| Validation TL | 25 % |
| Release note | 15 % |

Le score est indicatif.

Une anomalie critique doit rester visible indépendamment du score.

---

# 22. Dashboard HTML

Technologie recommandée :

- Jinja2 pour le templating ;
- HTML/CSS ;
- JavaScript léger ;
- pas de backend web dans le POC.

Le fichier final doit pouvoir être ouvert directement dans un navigateur :

```text
output/report.html
```

---

# 23. Structure du dashboard manager

Le dashboard doit comporter quatre niveaux.

## Niveau 1 — Executive Summary

Exemple :

```text
RELEASE QUALITY BOARD
September 2026

MAKS       91 %
GATEWAY    86 %
PKEM       92 %

GLOBAL     89 %

100 Jira
89 conformes
11 anomalies
```

Afficher quatre quality gates :

```text
PR COVERAGE
97 %

TL VALIDATION
98 %

RELEASE NOTES
93 %

PROCESS COMPLIANCE
89 %
```

---

## Niveau 2 — Timeline des releases

Afficher les différentes MEP du mois.

Exemple :

```text
September 2026

MAKS
04/09 ●
11/09 ●
18/09 ●
25/09 ●

GATEWAY
08/09 ●
17/09 ●
24/09 ●

PKEM
03/09 ●
10/09 ●
21/09 ●
29/09 ●
```

Chaque release doit être cliquable.

---

## Niveau 3 — Vue équipe

Pour chaque équipe :

```text
MAKS
32 Jira
29 conformes
3 anomalies

PR coverage       97 %
TL validation    100 %
Release notes     94 %
Process           91 %
```

Même logique pour GATEWAY et PKEM.

---

## Niveau 4 — Détail Jira

Exemple :

```text
MAKS-1245

Release: MAKS-2026.09.04
MEP: 04/09/2026
Developer: John Doe

PROCESS
New                 PASS
Analyzed            PASS
Ready to Dev        PASS
Code Review         FAIL

TL CHECK
Tech Lead           Jane Doe
Estimation          PASS
Golden Rules        PASS
Services impacted   PASS

GIT
Pull Request        MISSING
Release note        PASS
```

---

# 24. Section "Attention Required"

Le dashboard doit afficher en priorité les anomalies nécessitant une action.

Exemple :

```text
ATTENTION REQUIRED

MAKS-1245
PR manquante

MAKS-1290
Ready to Dev sans validation TL

GATEWAY-4521
PR renseignée mais inexistante

PKEM-451
Release note absente
```

---

# 25. UX du dashboard

Prévoir :

- filtres par équipe ;
- filtres par release ;
- filtres PASS/WARNING/FAIL ;
- recherche par Jira ;
- sections dépliables ;
- liens vers Jira ;
- liens vers PR ;
- liens vers release notes si disponibles.

Le dashboard doit être compréhensible en moins d'une minute par un manager.

---

# 26. Mode mock

Le POC doit pouvoir fonctionner sans accès réseau.

Commande :

```bash
python main.py --mock
```

Fixtures :

```text
tests/fixtures/
├── jira_release.json
├── jira_issue.json
├── jira_changelog.json
├── git_pr.json
└── release_notes.json
```

Objectif :

- développer les règles indépendamment des APIs ;
- tester les cas d'erreur ;
- faciliter les tests automatisés.

---

# 27. CLI cible

Prévoir une CLI simple.

Exemples :

```bash
python main.py --mock
```

```bash
python main.py --team MAKS --release "MAKS-2026.09.04"
```

```bash
python main.py --month 2026-09
```

```bash
python main.py --month 2026-09 --team MAKS
```

Options possibles :

```text
--mock
--team
--release
--month
--output
--debug
```

---

# 28. Gestion des secrets

Ne jamais stocker :

- token Jira ;
- token Git ;
- mot de passe ;
- secret API

dans `config.yml`.

Prévoir des variables d'environnement :

```text
JIRA_URL
JIRA_USERNAME
JIRA_TOKEN

GIT_URL
GIT_TOKEN
```

ou un mécanisme de secret manager compatible avec l'environnement bancaire.

---

# 29. Gestion des erreurs

Le programme doit distinguer :

### Erreur technique

Exemple :

```text
Jira API unavailable
```

=> le rapport ne doit pas déclarer le Jira conforme.

Il doit indiquer :

```text
UNKNOWN / DATA_UNAVAILABLE
```

### Anomalie métier

Exemple :

```text
PR_MISSING
```

=> `FAIL`.

Ne jamais transformer une absence de donnée due à une erreur API en anomalie métier.

---

# 30. Tests

Prévoir des tests unitaires sur les règles métier.

Cas minimum :

### Analyzed

- label présent avant Ready to Dev => PASS ;
- absent => FAIL ;
- ajouté après Ready to Dev => FAIL.

### TL

- transition effectuée par TL => PASS ;
- transition effectuée par développeur => FAIL ;
- transition introuvable => FAIL ou UNKNOWN selon le contexte.

### PR

- PR présente et accessible => PASS ;
- absente => FAIL ;
- URL invalide => FAIL ;
- PR ne correspondant pas au Jira => FAIL.

### Release note

- fichier présent => PASS ;
- fichier absent => FAIL.

### Champs obligatoires

- tous présents => PASS ;
- au moins un absent => FAIL.

---

# 31. Traçabilité

Chaque résultat doit être explicable.

Pour une anomalie, conserver :

- Jira ;
- règle ;
- valeur observée ;
- valeur attendue ;
- timestamp si pertinent ;
- source de la donnée.

Exemple :

```text
Jira: MAKS-1245
Rule: TL_VALIDATION_INVALID
Observed: developer01
Expected: one of [john.doe, jane.doe]
Transition: New -> Ready to Dev
Timestamp: 2026-09-02T10:04:12
```

Ceci est particulièrement important dans un contexte bancaire.

---

# 32. Historisation des rapports

Structure cible :

```text
reports/
└── 2026/
    ├── 09/
    │   ├── index.html
    │   ├── MAKS-2026.09.04.html
    │   ├── GATEWAY-2026.09.08.html
    │   └── PKEM-2026.09.10.html
    └── 08/
        └── ...
```

L'historique permet ensuite de calculer l'évolution mensuelle.

---

# 33. Roadmap

## POC Phase 1

Jira uniquement :

- récupération d'une release ;
- récupération des Jira ;
- récupération de l'historique ;
- contrôle `Analyzed` ;
- contrôle TL ;
- contrôle des champs.

## POC Phase 2

Git :

- récupération PR ;
- validation URL ;
- correspondance Jira/PR ;
- merge status.

## POC Phase 3

Release notes :

- recherche automatique ;
- contrôle Jira ↔ release note.

## POC Phase 4

HTML :

- executive summary ;
- équipe ;
- release ;
- Jira ;
- anomalies.

## Phase 5

Industrialisation :

- 3 équipes ;
- mois complet ;
- exécution planifiée ;
- historique ;
- tendances ;
- notifications éventuelles.

---

# 34. Extensions futures

Contrôles possibles :

- PR créée avant la MEP ;
- PR mergée avant la MEP ;
- PR approuvée par un TL ;
- nombre de reviewers ;
- branche cible ;
- Jira ajouté à la release tardivement ;
- Jira retiré puis ré-ajouté à une release ;
- plusieurs PR pour un Jira ;
- Jira sans commit associé ;
- release note ajoutée après la MEP ;
- temps `Ready to Dev -> Code Review` ;
- temps `Code Review -> Done` ;
- taux de conformité par équipe ;
- évolution mensuelle.

---

# 35. Principes de conception

1. **Séparation des responsabilités**
   - API clients ;
   - modèles ;
   - règles métier ;
   - reporting.

2. **Configuration externalisée**
   - équipes ;
   - TL ;
   - statuts ;
   - champs Jira ;
   - conventions Git.

3. **Pas de secrets dans le code.**

4. **Pas de dépendance au réseau pour les tests.**

5. **Les anomalies métier doivent être distinguées des erreurs techniques.**

6. **Toutes les anomalies doivent être explicables et traçables.**

7. **Le HTML est une vue du modèle de données et ne contient pas de logique métier.**

8. **Le moteur de qualité doit pouvoir être réutilisé pour d'autres formats de sortie.**

---

# 36. Critères d'acceptation du POC

Le POC sera considéré comme réussi lorsqu'il sera capable de :

- récupérer une release Jira ;
- récupérer tous les Jira de cette release ;
- récupérer l'historique de chaque Jira ;
- détecter si `Analyzed` a été effectué ;
- identifier l'utilisateur ayant effectué `New -> Ready to Dev` ;
- déterminer si cet utilisateur est TL ;
- vérifier les champs obligatoires ;
- récupérer la PR renseignée ;
- vérifier l'existence de la PR dans Git ;
- vérifier la cohérence Jira/PR ;
- rechercher la release note ;
- produire les anomalies ;
- produire un HTML lisible ;
- afficher les résultats par équipe et par release ;
- fonctionner en mode mock ;
- être couvert par des tests unitaires sur les règles principales.

---

# 37. Résultat attendu du premier POC

Commande :

```bash
python main.py --team MAKS --release "MAKS-2026.09.04"
```

Résultat :

```text
output/
└── MAKS-2026.09.04/
    ├── report.html
    └── report.json
```

Le JSON sert de donnée structurée/audit et le HTML de restitution management.

Exemple synthétique :

```text
Release: MAKS-2026.09.04

Jira: 18
Compliant: 16
Warning: 1
Fail: 1

PR Coverage: 94%
TL Validation: 100%
Release Notes: 94%
Process Compliance: 89%

FAIL
MAKS-1245
PR_MISSING

WARNING
MAKS-1281
RELEASE_NOTE_MISSING
```

---

# 38. Recommandation finale

Le POC doit être conçu comme un **moteur de contrôle qualité de release**, et non comme un simple script de reporting.

Architecture logique :

```text
Jira API ──────────────┐
                       │
                       v
                 DATA COLLECTION
                       │
Git API ───────────────┤
                       │
Git release notes ─────┘
                       │
                       v
                  QUALITY ENGINE
                       │
              ┌────────┴────────┐
              │                 │
              v                 v
        QualityReport       Anomalies
              │
              v
          HTML REPORT
```

Le premier objectif est de prouver la valeur sur une release et une équipe. Une fois le modèle et les règles validés, l'extension aux trois équipes et au reporting mensuel doit être essentiellement une évolution de configuration et de collecte, pas une réécriture de l'outil.
