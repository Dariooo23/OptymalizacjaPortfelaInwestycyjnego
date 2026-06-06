---
tags:
  - mpsi
  - projekt
  - raport
wiki_type: analysis
last_updated: 2026-05-25
---

# Raport — Szkic Struktury Notebooka

Gotowa struktura do przeklejenia do Jupyter Notebooka. Każda sekcja = komórka
Markdown w Notebooku. Wypełnij treścią po zakończeniu implementacji.

---

## Struktura Notebooka

---

### Sekcja 1: Wprowadzenie i cel

```markdown
# Optymalizacja Portfela Inwestycyjnego: K-means + Algorytm Genetyczny

**Kurs:** Matematyczne Podstawy Sztucznej Inteligencji (MPSI)  
**Autor:** [imię nazwisko]  
**Data:** [data]

## Cel projektu

Celem projektu jest budowa optymalnego portfela inwestycyjnego z akcji
wchodzących w skład indeksu S&P 500. Metodologia składa się z trzech etapów:

1. **Klasteryzacja (k-means):** grupowanie spółek wg profilu ryzyko-zwrot (μ, σ)
   i wyłonienie reprezentantów klastrów
2. **Eksploracja przestrzeni (Monte Carlo):** losowe portfele jako baseline
   i wizualizacja granicy efektywnej Markowitza
3. **Optymalizacja (algorytm genetyczny):** wyznaczenie wag portfela
   maksymalizujących wskaźnik Sharpe'a

Zarówno k-means jak i algorytm genetyczny zaimplementowane **od zera** (NumPy),
bez użycia bibliotek ML.
```

---

### Sekcja 2: Opis danych

```markdown
## Dane

**Źródło:** Yahoo Finance (biblioteka `yfinance`)  
**Okres:** [start] — [end] ([N] lat, [D] dni giełdowych)  
**Spółki:** [N] spółek z [K] sektorów S&P 500

### Wybór spółek

Wybrano spółki z różnych sektorów, aby zapewnić zróżnicowanie profili
ryzyko-zwrot — kluczowe dla efektywnego klastrowania.

[Tabela z tickerami i sektorami]

### Przetwarzanie

- Logarytmiczne stopy zwrotu: $r_t = \ln(P_t / P_{t-1})$  
- Annualizacja: $\mu \times 252$, $\Sigma \times 252$, $\sigma \times \sqrt{252}$  
- Stopa wolna od ryzyka: $R_f = 4\%$ (US T-bills 2024)

### Statystyki podstawowe

[Tabela: ticker | μ (roczny) | σ (roczne) | Sharpe indywidualny]
```

---

### Sekcja 3: Metodologia

```markdown
## Metodologia

### 3.1 Klasteryzacja — k-means

Spółki reprezentowane jako punkty $(\mu_i, \sigma_i)$ w przestrzeni ryzyko-zwrot.
Przed klasteryzacją: standaryzacja (StandardScaler).

**Algorytm:**
1. Inicjalizacja k-means++
2. Iteracyjne przypisanie i aktualizacja centroidów
3. Wybór k metodą łokcia i silhouette score

**Wynik:** [k] klastrów, [k] reprezentantów — jeden na klaster.

[Wykres: scatter (μ, σ) z kolorami klastrów i zaznaczonymi centroidami]

### 3.2 Symulacja Monte Carlo

[N_sim] losowych portfeli z rozkładu Dirichlet(1) — równomierne pokrycie
sympleksu wag. Dla każdego portfela obliczono: $\mu_p$, $\sigma_p$, Sharpe.

**Baseline:** najlepszy portfel MC: $S_{MC} = $ [wartość]

[Wykres: scatter (σ, μ) kolorowany Sharpe'm]

### 3.3 Algorytm genetyczny

**Kodowanie chromosomu:** wektor wag $\mathbf{w} \in \mathbb{R}^k$,
$\sum w_i = 1$, $w_i \geq 0$ — wymuszane normalizacją po każdej operacji.

**Parametry:** `pop_size=[N]`, `n_gen=[G]`, `sigma_mut=[M]`, `cx_rate=[C]`

**Operatory:**
- Selekcja: turniejowa ($k_{tourn} = $ [wartość])
- Crossover: convex ($\mathbf{c} = \alpha \mathbf{p}_1 + (1-\alpha)\mathbf{p}_2$)
- Mutacja: Gaussowska ($\varepsilon \sim \mathcal{N}(0, \sigma_{mut}^2)$)
- Elitaryzm: top-[N_elites] bez modyfikacji
```

---

### Sekcja 4: Wyniki i wizualizacje

```markdown
## Wyniki

### Portfel optymalny GA

| Metryka | Wartość |
|---------|---------|
| Sharpe ratio | [wartość] |
| Oczekiwany zwrot μ | [wartość] % |
| Ryzyko σ | [wartość] % |

**Skład portfela:**
[Wykres: bar chart wag]

### Granica efektywna — MC vs GA

[Wykres główny: scatter MC + punkt GA + punkt MC_best]

### Zbieżność GA

[Wykres: max Sharpe per generacja + linia MC_best jako baseline]
```

---

### Sekcja 5: Analiza hiperparametrów

```markdown
## Analiza wpływu hiperparametrów GA

Przeprowadzono [N] eksperymentów zmieniając jeden parametr przy pozostałych
stałych (ceteris paribus).

[Tabela eksperymentów — z [[Eksperymenty]]]

### Wnioski

**pop_size:** [wniosek]  
**sigma_mut:** [wniosek]  
**n_elites:** [wniosek]  
**tourn_k:** [wniosek]
```

---

### Sekcja 6: Porównanie GA vs losowanie

```markdown
## Porównanie GA vs Monte Carlo

| | GA | Monte Carlo |
|-|-----|-------------|
| Sharpe | [wartość] | [wartość] |
| μp | [wartość] | [wartość] |
| σp | [wartość] | [wartość] |
| Liczba ocen fitness | pop_size × n_gen | N_sim |

**Dlaczego GA > MC:**
Monte Carlo próbkuje przestrzeń losowo — $N$ portfeli pokrywa $N$ losowych punktów.
GA kieruje poszukiwanie: każda generacja buduje nowe osobniki *w okolicach*
dotychczas najlepszych rozwiązań. Przy tej samej liczbie ocen fitness GA
eksploruje przestrzeń efektywniej.

Przy $N_{sim} \to \infty$ MC dąży do globalnego optimum (prawo wielkich liczb),
ale w praktyce ograniczone zasoby obliczeniowe faworyzują GA.
```

---

### Sekcja 7: Ograniczenia i wnioski

```markdown
## Ograniczenia modelu

1. **Survivorship bias:** analizowane spółki to te, które przeżyły do dzisiaj —
   przecenia historyczne zwroty całego rynku

2. **Stacjonarność:** model Markowitza zakłada stałe μ i Σ w czasie — w praktyce
   parametry zmieniają się wraz z warunkami rynkowymi

3. **Look-only-back:** portfel optymalny historycznie może nie być optymalny
   w przyszłości (brak out-of-sample validation)

4. **Only long:** ograniczenie $w_i \geq 0$ wyklucza short selling —
   zawęża przestrzeń poszukiwań

5. **Gaussowskie założenie:** log-returns tylko *zbliżone* do normalności —
   fat tails (ogony) w rzeczywistych danych nie są modelowane

## Możliwe rozszerzenia

- Wskaźnik Sortino zamiast Sharpe'a (penalizuje tylko ujemne odchylenia)
- Short selling: usunięcie ograniczenia $w_i \geq 0$
- Rolling window: optymalizacja portfela w czasie (rebalancing co miesiąc)
- Więcej spółek + regularyzacja (ograniczenie maksymalnej wagi jednego aktywa)

## Wnioski

[Uzupełnij po zakończeniu eksperymentów]
```
