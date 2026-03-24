# Politique interne produit

## Politique de traitement, d’ingestion et de classification fiscale des exports Shakepay

### Particuliers seulement — Québec et Canada

### Version 1.0

## 1. Objet

1.1. La présente politique établit les règles internes applicables au traitement, à l’ingestion, à la normalisation, à la déduplication et à la classification fiscale des exports Shakepay utilisés par le logiciel.

1.2. Elle vise exclusivement les dossiers de **particuliers**, à l’exclusion des contextes suivants :
a) entreprise ;
b) travail autonome ;
c) emploi ;
d) activité commerciale ;
e) fiducie ;
f) société.

1.3. La présente politique a pour finalité de permettre la production de chiffres fiscaux destinés aux déclarations de revenus applicables au **Canada** et au **Québec**.

## 2. Champ d’application

2.1. La présente politique s’applique aux fichiers CSV Shakepay des familles suivantes :
a) `cash_transactions_summary` ;
b) `crypto_transactions_summary` ;
c) `usd_transactions_summary`.

2.2. Elle s’applique aux versions de ces fichiers en :
a) anglais ;
b) français.

2.3. La présente politique s’applique à la fois :
a) au traitement transactionnel ;
b) à la production des écritures fiscales ;
c) à la conservation des données ;
d) à l’audit interne ;
e) au support produit.

## 3. Sources normatives et interprétatives

3.1. Pour le Québec, Revenu Québec indique que les cryptoactifs sont des **biens** et que les transactions effectuées au moyen de cryptoactifs sont considérées comme du **troc**. ([Revenu Québec][1])

3.2. Revenu Québec exige également, pour les contribuables visés, la production du formulaire **TP-21.4.39 Déclaration relative aux cryptoactifs**, notamment lorsqu’un contribuable possède, reçoit, aliène ou utilise des cryptoactifs dans une transaction. ([Revenu Québec][2])

3.3. Pour le Canada, la CRA indique que les utilisateurs de cryptoactifs doivent déclarer leurs gains ou pertes, tenir des registres suffisants et déterminer la valeur de leurs cryptoactifs pour fins fiscales. ([Canada][3])

3.4. La CRA indique également que certains montants, notamment la plupart des **gifts** et certains **windfalls**, ne sont pas imposables selon les faits applicables. ([Canada][4])

3.5. La documentation Shakepay fournie par l’utilisateur décrit notamment les programmes suivants :
a) récompense de référencement ;
b) bonus promotionnels de référencement ;
c) ShakingSats ;
d) cashback carte ;
e) bonification promotionnelle 3 % ;
f) ShakeSquad ;
g) intérêts sur dépôts CAD ;
h) Shake It Forward.

## 4. Principes directeurs

4.1. **Principe de priorité technique à l’anglais.**
Le logiciel doit utiliser l’anglais comme **langue canonique interne** pour :
a) les identifiants techniques ;
b) les noms de champs normalisés ;
c) les catégories d’événements ;
d) les codes internes ;
e) les règles de classification.

4.2. **Principe de compatibilité bilingue.**
Malgré la priorité technique à l’anglais, le logiciel doit pouvoir ingérer, comprendre et normaliser les exports Shakepay en anglais et en français.

4.3. **Principe d’unicité canonique.**
Toute donnée importée, quelle que soit sa langue source, doit être convertie vers une représentation interne unique.

4.4. **Principe de non-double comptage.**
Le logiciel ne doit jamais compter deux fois une même transaction lorsque celle-ci est importée dans les deux langues.

4.5. **Principe de prudence fiscale.**
Lorsqu’aucune position spécifique explicite de la CRA ou de Revenu Québec n’a été identifiée pour un programme Shakepay donné, le logiciel applique une position prudente documentée comme telle.

4.6. **Principe de traçabilité intégrale.**
Toute transaction traitée doit pouvoir être retracée :
a) à son fichier d’origine ;
b) à sa ligne brute ;
c) à sa version normalisée ;
d) à sa classification fiscale ;
e) à son empreinte de déduplication.

## 5. Langue canonique et conventions internes

5.1. La langue canonique interne du système est l’**anglais**.

5.2. Les identifiants internes doivent être exprimés en anglais, y compris notamment :
a) `timestamp` ;
b) `raw_type` ;
c) `raw_description` ;
d) `amount_debited` ;
e) `asset_debited` ;
f) `amount_credited` ;
g) `asset_credited` ;
h) `market_value_cad` ;
i) `book_cost_cad` ;
j) `normalized_event`.

5.3. Les catégories fiscales internes doivent être exprimées en anglais, y compris notamment :
a) `taxable_income` ;
b) `purchase_rebate` ;
c) `non_taxable_review` ;
d) `personal_non_deductible_outflow`.

5.4. Les libellés français sont admissibles à l’import, mais ne doivent pas être utilisés comme nomenclature canonique interne.

## 6. Fichiers admis

6.1. Le logiciel doit accepter les fichiers suivants, en anglais ou en français :
a) `cash_transactions_summary` ;
b) `crypto_transactions_summary` ;
c) `usd_transactions_summary`.

6.2. Les trois familles sont réputées **utiles** au fonctionnement du moteur.

6.3. Utilité normative des familles :
a) `cash_transactions_summary` : flux CAD, rewards en CAD, bonus de référencement, intérêts CAD, paiements carte, dépôts et retraits ;
b) `crypto_transactions_summary` : rewards en crypto, acquisitions, dispositions, envois, réceptions, valeurs de marché, coûts fiscaux ;
c) `usd_transactions_summary` : flux USD, conversions, réconciliations multi-actifs, opérations USD.

6.4. Le logiciel doit être conçu pour fonctionner même si une ou plusieurs familles sont absentes, mais il doit alors signaler la possibilité d’un dossier incomplet.

## 7. Utilité des colonnes

7.1. Toutes les colonnes reconnues doivent être conservées à l’import.

7.2. Les colonnes sont classées en trois niveaux.

### 7.2.1. Colonnes critiques

Sont des colonnes critiques :
a) date ;
b) type ;
c) description ;
d) montant débité ou débit ;
e) actif débité, si applicable ;
f) montant crédité ou crédit ;
g) actif crédité, si applicable ;
h) valeur marchande ;
i) devise de la valeur marchande.

### 7.2.2. Colonnes fortement recommandées

Sont fortement recommandées :
a) coût comptable ;
b) devise du coût comptable ;
c) taux spot ;
d) taux achat/vente.

### 7.2.3. Colonnes de validation et d’audit

Sont utiles pour validation :
a) solde ;
b) identifiants dérivés ;
c) métadonnées de source ;
d) signatures de rapprochement.

7.3. Aucun champ reconnu ne doit être supprimé lors de l’ingestion brute.

## 8. Dictionnaire canonique minimal

8.1. Les équivalences de colonnes doivent être normalisées au minimum comme suit :

* `Date` → `timestamp`
* `Type` → `raw_type`
* `Description` → `raw_description`
* `Debit` / `Débit` → `debit`
* `Credit` / `Crédit` → `credit`
* `Balance` / `Solde` → `balance`
* `Amount Debited` / `Montant débité` → `amount_debited`
* `Asset Debited` / `Actif débité` → `asset_debited`
* `Amount Credited` / `Montant crédité` → `amount_credited`
* `Asset Credited` / `Actif crédité` → `asset_credited`
* `Market Value` / `Valeur marchande` → `market_value`
* `Market Value Currency` / `Devise de la valeur marchande` → `market_value_currency`
* `Book Cost` / `Coût comptable` → `book_cost`
* `Book Cost Currency` / `Devise du coût comptable` → `book_cost_currency`
* `Spot Rate` / `Taux au comptant` → `spot_rate`
* `Buy / Sell Rate` / `Taux d'achat / vente` → `buy_sell_rate`

8.2. Toute nouvelle variation d’en-tête compatible doit être mappée vers ce schéma canonique.

## 9. Politique d’ingestion bilingue

9.1. Le logiciel doit accepter les imports en anglais et en français.

9.2. Le logiciel doit déterminer automatiquement, pour chaque fichier :
a) sa famille ;
b) sa langue ;
c) sa validité structurelle ;
d) sa compatibilité avec le schéma attendu.

9.3. L’ingestion bilingue ne doit pas modifier les règles fiscales appliquées. Seule la langue de surface change ; la classification interne demeure identique.

9.4. En cas d’import de deux fichiers de même famille dans les deux langues, le logiciel doit :
a) ingérer les deux ;
b) comparer les transactions ;
c) dédupliquer les écritures économiques identiques ;
d) conserver les sources pour audit.

## 10. Politique de déduplication interlangue

10.1. Deux lignes provenant de langues différentes doivent être considérées comme représentant une même transaction lorsqu’elles correspondent, à une tolérance raisonnable, sur les éléments suivants :
a) date/heure ;
b) type normalisé ;
c) description normalisée ;
d) actif débité ;
e) actif crédité ;
f) montant débité ;
g) montant crédité ;
h) valeur marchande ;
i) coût comptable, si présent.

10.2. Lorsqu’un doublon interlangue est détecté, une seule transaction logique doit être retenue pour le calcul fiscal.

10.3. Les deux versions sources doivent néanmoins être conservées à des fins :
a) d’audit ;
b) de support ;
c) de comparaison ;
d) de résolution d’anomalies.

10.4. En cas de divergence matérielle entre deux lignes supposées équivalentes, la fusion automatique est interdite. La transaction doit être placée en revue.

## 11. Classification des événements Shakepay

11.1. Le logiciel doit classifier les événements reconnus dans une nomenclature canonique anglaise.

11.2. Les catégories minimales suivantes doivent être supportées :
a) `shakingsats` ;
b) `referral_bonus` ;
c) `referral_promo_bonus` ;
d) `cad_interest` ;
e) `shakesquad_reward` ;
f) `card_cashback_btc` ;
g) `card_intro_bonus_3pct` ;
h) `sif_received` ;
i) `sif_sent` ;
j) `buy` ;
k) `sell` ;
l) `send` ;
m) `receive`.

11.3. Les libellés anglais et français doivent être mappés vers ces identifiants uniques.

## 12. Politique fiscale de classification — particulier seulement

### 12.1. Événements inclus au revenu

Doivent être inclus au revenu, par défaut :
a) `shakingsats` ;
b) `referral_bonus` ;
c) `referral_promo_bonus` ;
d) `shakesquad_reward` ;
e) `cad_interest`.

12.1.1. Lorsque l’actif reçu est un cryptoactif, la juste valeur marchande au moment de la réception doit également être inscrite comme coût d’acquisition de l’actif reçu. Cette approche s’aligne sur l’exigence générale de valorisation et de tenue de registres applicable aux cryptoactifs. ([Canada][3])

### 12.2. Événements traités comme rabais de consommation

Doivent être classés par défaut comme `purchase_rebate` et non comme revenu ordinaire :
a) `card_cashback_btc` ;
b) `card_intro_bonus_3pct`.

12.2.1. Pour ces événements, la crypto reçue doit néanmoins être inscrite au registre des acquisitions avec sa valeur d’entrée.

12.2.2. Cette position constitue une **position prudente de politique produit**. Je n’ai pas trouvé de position spécifique publiée par la CRA ou Revenu Québec sur le cashback crypto Shakepay lui-même. Elle repose donc sur une qualification économique prudente, non sur une règle explicite dédiée.

### 12.3. Événements distincts non imposables par défaut, sous revue

`Shake It Forward` reçu doit être classé dans une catégorie distincte :
a) `sif_received` → `non_taxable_review`.

12.3.1. Cette position s’appuie sur la description Shakepay du programme comme contribution volontaire, finale, non remboursable et sans rendement financier, ainsi que sur les principes généraux de la CRA relatifs aux gifts et windfalls.  ([Canada][4])

12.3.2. Cette position est prudente et doit être marquée comme telle.

### 12.4. Sorties personnelles non déductibles

`Shake It Forward` envoyé doit être classé comme :
a) `sif_sent` → `personal_non_deductible_outflow`.

## 13. Ordre normatif de priorité de classification

13.1. En cas d’ambiguïté de libellé, le moteur doit appliquer l’ordre suivant :
a) `sif` ;
b) `interest` ;
c) `referral` ;
d) `shakesquad` ;
e) `shakingsats` ;
f) `card_cashback` ;
g) `unknown_reward_review`.

13.2. Cette hiérarchie vise à réduire les erreurs de classification par absorption dans une catégorie trop générale.

## 14. Règles opérationnelles

14.1. Pour tout événement classé `taxable_income`, le moteur doit :
a) inclure le montant dans le revenu ;
b) si la réception est en crypto, créer une entrée de coût d’acquisition ;
c) conserver la source et la justification de classification.

14.2. Pour tout événement classé `purchase_rebate`, le moteur doit :
a) exclure le montant du revenu par défaut ;
b) créer une entrée de coût d’acquisition de la crypto reçue ;
c) conserver une trace distincte du rabais.

14.3. Pour tout événement classé `non_taxable_review`, le moteur doit :
a) exclure le montant du revenu par défaut ;
b) générer un drapeau de revue ;
c) conserver le motif de prudence.

14.4. Pour tout événement classé `personal_non_deductible_outflow`, le moteur doit :
a) ne créer aucune déduction ;
b) conserver l’écriture à titre informatif et d’audit.

## 15. Dossier incomplet

15.1. Le logiciel doit signaler un dossier comme potentiellement incomplet lorsque :
a) une famille de fichier habituellement pertinente manque ;
b) des colonnes critiques sont absentes ;
c) des doublons non résolus persistent ;
d) des lignes non classifiées subsistent ;
e) les soldes ou flux sont incohérents.

15.2. Le logiciel ne doit pas affirmer qu’un dossier est fiscalement complet sans vérification de cohérence suffisante.

## 16. Conservation et audit

16.1. Le logiciel doit conserver :
a) le fichier brut ;
b) la langue détectée ;
c) la famille détectée ;
d) les lignes brutes ;
e) les lignes normalisées ;
f) la décision de classification ;
g) les drapeaux de revue ;
h) l’empreinte de déduplication.

16.2. Toute décision automatisée importante doit être audit able a posteriori.

## 17. Portée des positions prudentes

17.1. Les positions suivantes sont des **positions prudentes de politique produit**, et non des affirmations de droit positif spécifique à Shakepay :
a) `card_cashback_btc` comme rabais de consommation ;
b) `card_intro_bonus_3pct` comme rabais de consommation ;
c) `sif_received` comme avantage non imposable par défaut sous revue.

17.2. Ces positions doivent pouvoir être modifiées par mise à jour normative du moteur si une publication officielle spécifique de la CRA ou de Revenu Québec venait à les contredire ou à les préciser.

## 18. Mise à jour de la politique

18.1. La présente politique doit être revue lorsqu’un des événements suivants survient :
a) modification des exports Shakepay ;
b) ajout d’un nouveau programme de reward ;
c) changement majeur de structure CSV ;
d) publication officielle pertinente de la CRA ;
e) publication officielle pertinente de Revenu Québec.

18.2. Toute mise à jour doit être versionnée.

## 19. Décision produit officielle

19.1. Le logiciel doit être **bilingue en entrée**.

19.2. Le logiciel doit être **anglais en interne**.

19.3. Tous les fichiers reconnus sont **utiles**.

19.4. Toutes les colonnes reconnues sont **à conserver**, même si leur criticité diffère.

19.5. Les fichiers anglais et français d’une même famille ne doivent **jamais** produire un double comptage.

19.6. Les catégories fiscales doivent suivre la présente politique tant qu’aucune politique versionnée plus récente ne la remplace.

## 20. Annexe A — Décision synthétique

20.1. Inclus au revenu :

* `shakingsats`
* `referral_bonus`
* `referral_promo_bonus`
* `shakesquad_reward`
* `cad_interest`

20.2. Exclus du revenu par défaut, mais suivis comme acquisitions crypto :

* `card_cashback_btc`
* `card_intro_bonus_3pct`

20.3. Exclus du revenu par défaut, avec revue :

* `sif_received`

20.4. Non déductibles :

* `sif_sent`

## 21. Annexe B — Décision sur la priorité de l’anglais

21.1. Pour des fins de simplicité, de maintenance, de documentation technique, de test, de validation et d’interopérabilité, l’anglais est la langue normative du modèle interne.

21.2. Le français est une langue de compatibilité d’ingestion et d’interface, mais non la langue canonique des identifiants internes.

21.3. Cette décision n’affecte ni la validité du traitement des fichiers français ni la capacité du produit à fonctionner en français côté utilisateur.

[1]: https://www.revenuquebec.ca/fr/une-mission-des-actions/vous-aider-a-vous-conformer/quest-ce-que-leconomie-numerique/les-cryptoactifs/?utm_source=chatgpt.com "Les cryptoactifs"
[2]: https://www.revenuquebec.ca/fr/services-en-ligne/formulaires-et-publications/details-courant/tp-21-4-39/?utm_source=chatgpt.com "Déclaration relative aux cryptoactifs TP-21.4.39"
[3]: https://www.canada.ca/en/revenue-agency/programs/about-canada-revenue-agency-cra/compliance/cryptocurrency-guide/crypto-assets-tax-obligations.html?utm_source=chatgpt.com "Understanding crypto-assets and your tax obligations"
[4]: https://www.canada.ca/en/revenue-agency/services/tax/individuals/topics/about-your-tax-return/tax-return/completing-a-tax-return/personal-income/amounts-that-taxed.html?utm_source=chatgpt.com "Amounts that are not reported or taxed"
