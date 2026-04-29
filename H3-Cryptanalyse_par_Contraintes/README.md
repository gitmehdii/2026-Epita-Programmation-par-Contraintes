# H3 - Cryptanalyse par Contraintes (Ciphers Avancés)

> Utiliser la programmation par contraintes (CP-SAT) pour casser des chiffrements classiques : substitution monoalphabétique, Vigenère, transposition et Hill cipher.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Architecture du projet](#2-architecture-du-projet)
3. [Contraintes linguistiques implémentées](#3-contraintes-linguistiques-implémentées)
4. [Chiffrement par substitution — H3-1](#4-chiffrement-par-substitution--h3-1)
5. [Chiffrement de Vigenère — H3-2](#5-chiffrement-de-vigenère--h3-2)
6. [Chiffrement par transposition — H3-3](#6-chiffrement-par-transposition--h3-3)
7. [Hill Cipher — H3-4](#7-hill-cipher--h3-4)
8. [Évaluation comparée — H3-5](#8-évaluation-comparée--h3-5)
9. [Installation et utilisation](#9-installation-et-utilisation)
10. [Avancement](#10-avancement)

---

## 1. Vue d'ensemble

### Problématique

La cryptanalyse classique repose sur l'**analyse statistique** (fréquences de lettres, bigrammes). L'approche par **programmation par contraintes** encode les propriétés du chiffrement et du langage naturel directement comme des contraintes CSP, permettant une recherche systématique et combinable dans l'espace des clés possibles.

**Avantages de l'approche CP :**
- Combine simultanément plusieurs sources d'information (fréquences, bigrammes, trigrammes, mots connus)
- Chaque connaissance supplémentaire devient une contrainte qui réduit l'espace de recherche
- Garantit l'optimalité de la solution trouvée (preuve formelle)
- Se généralise naturellement à des chiffrements plus complexes

### Ce que le projet n'est pas

Ce projet n'est pas de la cryptanalyse par machine learning ni de la force brute aveugle. La force vient de l'**encodage symbolique des contraintes** : chaque règle du langage et du chiffrement devient une contrainte CSP.

### Résultats obtenus

| Chiffrement | Espace clé | Approche | Précision | Temps |
|-------------|-----------|----------|-----------|-------|
| Substitution (+ mot connu) | 26! ≈ 4×10²⁶ | CP-SAT | ~95–100% | < 5 s |
| Substitution (pur) | 26! ≈ 4×10²⁶ | Hill climbing + trigrammes | ~90% (400+ lettres) | < 2 s |
| Vigenère | 26^L | IC → CP-SAT | 100% (8/8 clés) | 4–6 s |

---

## 2. Architecture du projet

```
H3-Cryptanalyse_par_Contraintes/
│
├── README.md
├── requirements.txt
│
├── notebooks/                              ← notebooks Jupyter à exécuter dans l'ordre
│   ├── H3-1-Substitution.ipynb            [✓] Substitution mono — 3 approches + benchmark
│   ├── H3-2-Vigenere.ipynb                [✓] Vigenère — IC + Kasiski + CP-SAT
│   ├── H3-3-Transposition.ipynb           [✓] Transposition — CP-SAT + benchmark
│   ├── H3-4-Hill.ipynb                    [✓] Hill cipher — connu + seul + benchmark
│   └── H3-5-Evaluation.ipynb              [✓] Benchmark comparatif global
│
├── core/
│   ├── ciphers/
│   │   ├── substitution.py                [✓] encrypt / decrypt / key_accuracy
│   │   ├── vigenere.py                    [✓] encrypt / decrypt / str_to_key / key_to_str
│   │   ├── transposition.py               [✓] encrypt / decrypt / generate_random_key
│   │   └── hill.py                        [✓] encrypt / decrypt / known_plaintext_attack
│   ├── solvers/
│   │   ├── cp_substitution.py             [✓] CP-SAT : AllDifferent + coûts bigrammes + unigrammes
│   │   ├── cp_vigenere.py                 [✓] CP-SAT : agrégation par paire de positions-clé
│   │   ├── hill_climbing.py               [✓] Hill climbing bigrammes/trigrammes + restarts
│   │   ├── cp_transposition.py            [✓] CP-SAT : AllDifferent + table agrégée within-row
│   │   └── cp_hill.py                     [✓] CP-SAT : connu (mod 26) + seul (bigrammes)
│   ├── linguistics/
│   │   └── frequency_analysis.py          [✓] IC, Kasiski, bigrammes, trigrammes, freq attack
│   └── evaluation/
│       └── benchmark.py                   [✓] run_trials, print_table, compare_approaches
│
└── data/
    ├── french_reference.txt               [✓] 8 314 lettres de référence
    ├── french_bigrams_standard.json       [✓] 676 bigrammes avec log-probabilités
    └── french_trigrams_standard.json      [✓] 17 576 trigrammes avec log-probabilités
```

---

## 3. Contraintes linguistiques implémentées

### 3.1 Indice de coïncidence (IC)

```python
IC = Σ f(l)×(f(l)−1) / (n×(n−1))
```

| Langue | IC typique |
|--------|-----------|
| Français | ~0.065 |
| Anglais | ~0.061 |
| Texte chiffré Vigenère | ~0.038–0.055 |
| Texte uniforme (aléatoire) | ~0.038 |

L'IC est **conservé par une substitution monoalphabétique** (clé fixe) — signal clé pour distinguer les types de chiffrement.

### 3.2 Score de bigrammes (fonction objectif CP-SAT)

```python
score = Σ log P(texte_clair[i:i+2])   pour tout i
```

Encodé dans CP-SAT via `AddElement` sur une table de coûts entière :

```python
cost_table[a*26+b] = round(-log_prob('AB') × 1000)
# Faible = bigramme fréquent en français (ES, EN, LE...)
# Élevé  = bigramme rare/impossible (ZX, WK...)

model.add_element(bigram_idx, cost_table, cost_var)
model.minimize(sum(cost_vars))
```

### 3.3 Score de trigrammes

Utilisé par le hill climbing pour une meilleure discrimination (les bigrammes seuls peuvent ne pas distinguer JUSTICE de QUSTIDE car QU, US, ST sont tous fréquents).

### 3.4 Coûts unigrammes (guide de recherche CP-SAT)

Pour la substitution, on ajoute un coût proportionnel à la distance entre le rang de fréquence de la lettre chiffrée et celui de la lettre claire proposée :

```python
cost_unigram[c] = UNIGRAM_SCALE × |rang_freq(c) − rang_freq_fr(key[c])|
```

26 contraintes `AddElement` supplémentaires qui guident le solveur vers des correspondances cohérentes avec les fréquences.

---

## 4. Chiffrement par substitution — H3-1

### Principe

Chaque lettre du texte clair est remplacée par une lettre fixe. La clé est une **permutation de l'alphabet** (bijection 26→26).

```
Plain  : A B C ... E ... L  A  J  U  S  T  I  C  E ...
Cipher : X Q W ... R ... Y  X  O  Z  K  P  N  C  R ...
```

**Espace** : 26! ≈ 4×10²⁶ — infaisable en force brute.

### Modèle CP-SAT

```
Variables   : key[i] ∈ [0..25] pour i ∈ [0..25]
              key[i] = j  ↔  lettre chiffrée i déchiffrée en lettre j
Contrainte  : AllDifferent(key)
Objectif    : minimize Σ count(c1,c2) × cost_table[key[c1]×26 + key[c2]]
            + Σ UNIGRAM_SCALE × |rang(c) − rang_fr(key[c])|
```

### Trois approches comparées

| Approche | Précision (400 lettres) | Temps | Garantie |
|----------|------------------------|-------|----------|
| Analyse de fréquence | ~40–60% | < 1 ms | Non |
| Hill climbing (trigrammes) | ~90% | 0.5–2 s | Non (local opt.) |
| CP-SAT + mot connu | ~95–100% | < 5 s | Optimale |

### Force du CP-SAT : les contraintes additionnelles

Quand on connaît un mot du texte clair (ex : "JUSTICE"), on fixe directement les 7 lettres correspondantes :

```python
for p, c in zip("JUSTICE", cipher_of_JUSTICE):
    model.add(key[ord(c)-65] == ord(p)-65)
```

L'espace passe de 26! à 19! — CP-SAT trouve la solution en quelques secondes.

---

## 5. Chiffrement de Vigenère — H3-2

### Principe

La clé est un mot de longueur L. Chaque lettre est décalée par la valeur correspondante de la clé (cycliquement).

```
Clé     :  F   R   A   N   C   E   F   R   A ...
Décalage:  5  17   0  13   2   4   5  17   0 ...
Clair   :  L   A   J   U   S   T   I   C   E ...
Chiffré :  Q   R   J   H   U   X   N   T   E ...
```

`chiffré[i] = (clair[i] + clé[i mod L]) mod 26`

### Pipeline d'attaque

```
1. IC + Kasiski → longueurs candidates {L1, L2, ...}
2. CP-SAT(L1)  → clé candidate + réduction de période
3. Si échec    → CP-SAT(L2), etc.
```

### Modèle CP-SAT (innovation : agrégation par paires)

```
Variables : key[j] ∈ [0..25] pour j ∈ [0..L-1]

Pour chaque paire de positions-clé (j1, j2) :
  agg_cost[a*26+b] = Σ cost_bigram[((ci−a+26)%26)×26 + ((ci1−b+26)%26)]
                     somme sur toutes les positions (i, i+1) où (i%L, (i+1)%L) = (j1, j2)

→ L² contraintes AddElement au total (36 pour L=6)
```

### Résultats

| Clé | L | Détection IC | CP-SAT | Temps |
|-----|---|-------------|--------|-------|
| CLEF | 4 | ✓ | ✓ exact | 3.7 s |
| PARIS | 5 | ✓ | ✓ exact | 4.1 s |
| FRANCE | 6 | ✓ | ✓ exact | 4.0 s |
| NAPOLEON | 8 | ✓ | ✓ exact | 4.4 s |
| REPUBLIQUE | 10 | ✓ | ✓ exact | 5.2 s |

---

## 6. Chiffrement par transposition — H3-3

### Principe

Les lettres ne sont **pas substituées** mais **réarrangées** selon une permutation de colonnes.

```
Clair   : BONJOUR PARIS  →  en colonnes de largeur L=4
Matrice :  B O N J
           O U R P
           A R I S
Clé     : [3, 1, 4, 2]   (ordre de lecture des colonnes)
Chiffré : lecture dans l'ordre de la clé
```

### Modèle CP-SAT prévu

```
Variables   : perm[j] ∈ [0..L-1] pour j ∈ [0..L-1]
Contrainte  : AllDifferent(perm)
Objectif    : minimize coût bigrammes du texte reconstitué selon perm
```

**Statut** : *à implémenter* (`H3-3-Transposition.ipynb`)

---

## 7. Hill Cipher — H3-4

### Principe

Chiffrement **linéaire par blocs** — multiplication matricielle mod 26 sur des blocs de n lettres.

```
K = [[6, 24], [1, 13]]   (clé : matrice 2×2 inversible mod 26)
[c1, c2]ᵀ = K × [p1, p2]ᵀ  mod 26
```

### Modèle CP-SAT prévu

```
Variables        : K[i][j] ∈ [0..25]  (entrées de la matrice clé)
Contrainte forte : gcd(det(K) mod 26, 26) = 1  (inversibilité)
Objectif         : bigrammes des blocs déchiffrés ≈ français
```

**Cas d'usage CP-SAT :** attaque à texte clair connu partiel — quelques paires (clair, chiffré) suffisent à résoudre le système linéaire mod 26.

**Statut** : *à implémenter* (`H3-4-Hill.ipynb`)

---

## 8. Évaluation comparée — H3-5

### Métriques

| Métrique | Description |
|----------|-------------|
| `key_accuracy` | % de lettres/valeurs de clé correctement trouvées |
| `plaintext_score` | Score bigramme log-vraisemblance du texte déchiffré |
| `time_s` | Temps de résolution (secondes) |
| `success_rate` | % d'essais avec clé exacte |

### Protocole prévu

1. 10 clés aléatoires × 5 longueurs de texte (100 à 1000 lettres) × 4 chiffrements
2. Approches : fréquence, hill climbing, CP-SAT pur, CP-SAT + contraintes
3. Courbes `success_rate` vs `text_length` par approche

**Statut** : *à implémenter* (`H3-5-Evaluation.ipynb`)

---

## 9. Installation et utilisation

### Dépendances

```bash
pip install -r requirements.txt
# → ortools, numpy, matplotlib, jupyter
```

### Lancement des notebooks

```bash
cd H3-Cryptanalyse_par_Contraintes
jupyter notebook notebooks/
```

### Utilisation directe des modules

```python
from core.ciphers.substitution import generate_random_key, encrypt
from core.linguistics.frequency_analysis import bigram_log_probs, trigram_log_probs
from core.solvers.hill_climbing import hill_climbing_attack
from core.solvers.cp_substitution import solve_substitution
from core.ciphers.vigenere import encrypt as v_encrypt
from core.solvers.cp_vigenere import solve_vigenere

# Substitution
with open('data/french_reference.txt') as f:
    corpus = f.read()
tlp = trigram_log_probs(corpus)
blp = bigram_log_probs(corpus)

key = generate_random_key()
cipher = encrypt("BONJOUR MONDE", key)
result = hill_climbing_attack(cipher, tlp, letter_frequencies(corpus), ngram_size=3)

# Vigenère
from core.linguistics.frequency_analysis import detect_key_length_ic, letter_frequencies
cipher_v = v_encrypt("BONJOUR MONDE", "FRANCE")
L = detect_key_length_ic(cipher_v)[0][0]
res = solve_vigenere(cipher_v, L, blp)
print(res['key_str'])  # → FRANCE
```

### Ordre des notebooks

| Notebook | Contenu | Statut |
|----------|---------|--------|
| `H3-1-Substitution.ipynb` | Substitution mono, 3 approches, benchmark | ✓ Terminé |
| `H3-2-Vigenere.ipynb` | IC, Kasiski, CP-SAT, pipeline complet | ✓ Terminé |
| `H3-3-Transposition.ipynb` | Transposition, permutation CP-SAT | ✓ Terminé |
| `H3-4-Hill.ipynb` | Hill cipher, algèbre mod 26 | ✓ Terminé |
| `H3-5-Evaluation.ipynb` | Benchmark comparatif global | ✓ Terminé |

---

## 10. Avancement

### Terminé ✓

- [x] **H3-1 — Substitution monoalphabétique**
  - [x] `core/ciphers/substitution.py` — encrypt, decrypt, key_accuracy
  - [x] `core/solvers/cp_substitution.py` — AllDifferent + coûts bigrammes + coûts unigrammes + hints
  - [x] `core/solvers/hill_climbing.py` — bigrammes/trigrammes, restarts aléatoires
  - [x] `notebooks/H3-1-Substitution.ipynb` — 3 approches + CP-SAT avec mots connus + benchmark
- [x] **H3-2 — Vigenère**
  - [x] `core/ciphers/vigenere.py` — encrypt, decrypt, réduction de période
  - [x] `core/linguistics/frequency_analysis.py` — IC, Kasiski, detect_key_length_ic
  - [x] `core/solvers/cp_vigenere.py` — agrégation par paires, L² contraintes, 8/8 clés exactes
  - [x] `notebooks/H3-2-Vigenere.ipynb` — IC, Kasiski, CP-SAT, benchmark multi-clés
- [x] **Données linguistiques**
  - [x] `data/french_reference.txt` — 8 314 lettres de référence
  - [x] `data/french_bigrams_standard.json` — 676 bigrammes
  - [x] `data/french_trigrams_standard.json` — 17 576 trigrammes

### Terminé ✓ (suite)

- [x] **H3-3 — Transposition columnar**
  - [x] `core/ciphers/transposition.py` — encrypt, decrypt, generate_random_key, key_accuracy
  - [x] `core/solvers/cp_transposition.py` — AllDifferent + table agrégée within-row, L-1 contraintes
  - [x] `notebooks/H3-3-Transposition.ipynb` — IC conservé, CP-SAT, benchmark L=4..8
- [x] **H3-4 — Hill cipher**
  - [x] `core/ciphers/hill.py` — encrypt, decrypt, known_plaintext_attack, _matrix_inv_mod26
  - [x] `core/solvers/cp_hill.py` — attaque connue (linéaire mod 26) + attaque seule (bigrammes)
  - [x] `notebooks/H3-4-Hill.ipynb` — algèbre, CP-SAT connu (100%), CP-SAT seul
- [x] **H3-5 — Évaluation comparée**
  - [x] `core/evaluation/benchmark.py` — run_trials, print_table, compare_approaches
  - [x] `notebooks/H3-5-Evaluation.ipynb` — benchmark global 4 chiffrements + courbes
