# Teoria — Optymalizacja Portfela

Podkład matematyczny dla projektu. Wszystkie wzory bezpośrednio wykorzystywane
w implementacji. Nawiązania do wykładów MPSI oznaczone przy każdej sekcji.

---

## 1. Wskaźnik Sharpe'a

$$
S = \frac{E(R_p) - R_f}{\sigma_p}
$$

| Symbol | Znaczenie |
|--------|-----------|
| $E(R_p)$ | oczekiwana roczna stopa zwrotu portfela |
| $R_f$ | stopa wolna od ryzyka (np. bony skarbowe USA ≈ 4% w 2024) |
| $\sigma_p$ | roczne odchylenie standardowe stopy zwrotu portfela |

**Interpretacja:** liczba jednostek nadwyżki zwrotu (ponad $R_f$) na jednostkę ryzyka.

- $S > 1$ — portfel dobrze wynagradzający ryzyko
- $S \in (0, 1)$ — portfel opłacalny, ale słabo
- $S < 0$ — portfel gorszy od bezpiecznej alternatywy (np. obligacji)

**Funkcja celu projektu:** $\max_w S(w)$ przy ograniczeniach $\sum w_i = 1$, $w_i \geq 0$.

---

## 2. μ i σ portfela z macierzy kowariancji

*(nawiązanie do [[04 - Probabilistyczne Podstawy ML]])*

### Wektor wag

$$
\mathbf{w} = [w_1, w_2, \ldots, w_n]^\top, \quad \sum_{i=1}^n w_i = 1, \quad w_i \geq 0
$$

### Oczekiwany zwrot portfela

$$
\mu_p = \mathbf{w}^\top \boldsymbol{\mu} = \sum_{i=1}^n w_i \mu_i
$$

gdzie $\boldsymbol{\mu} = [\mu_1, \ldots, \mu_n]^\top$ to wektor oczekiwanych zwrotów poszczególnych aktywów.

### Wariancja portfela

$$
\sigma_p^2 = \mathbf{w}^\top \boldsymbol{\Sigma} \mathbf{w}
$$

$$
\sigma_p^2 = \sum_{i=1}^n \sum_{j=1}^n w_i w_j \Sigma_{ij}
$$

### Macierz kowariancji

$$
\Sigma_{ij} = \text{Cov}(R_i, R_j) = E\left[(R_i - \mu_i)(R_j - \mu_j)\right]
$$

- $\Sigma_{ii} = \sigma_i^2$ (wariancja aktywu $i$)
- $\Sigma_{ij} = \rho_{ij} \cdot \sigma_i \cdot \sigma_j$ gdzie $\rho_{ij} \in [-1, 1]$ to korelacja

**Kluczowa właściwość — dywersyfikacja:** gdy $\rho_{ij} < 1$, portfel ma mniejsze $\sigma_p$ niż średnia ważona $\sigma_i$. Im niższe korelacje między aktywami, tym większy efekt dywersyfikacji.

**Skąd bierze się $\Sigma$ w projekcie:**

```python
returns = np.log(prices / prices.shift(1)).dropna()
mu = returns.mean() * 252          # annualizacja (252 dni giełdowe)
cov = returns.cov() * 252          # macierz kowariancji annualizowana
sigma_p = np.sqrt(w @ cov @ w)    # odchylenie standardowe portfela
```

---

## 3. Granica efektywna Markowitza

**Definicja:** zbiór portfeli o minimalnym ryzyku ($\sigma_p$) dla każdego zadanego poziomu zwrotu ($\mu_p$). Portfele leżące poniżej granicy są zdominowane — istnieje portfel z takim samym ryzykiem i wyższym zwrotem.

**Capital Market Line (CML):** prosta łącząca punkt $R_f$ z portfelem rynkowym $M$ (tangencyjnym do granicy efektywnej). Portfele na CML są optymalne dla inwestorów zakładających pożyczki/lokaty po stopie $R_f$.

**Portfel maksymalizujący Sharpe'a:** punkt, w którym CML jest styczna do granicy efektywnej. Jest to portfel rynkowy $M$, który GA szuka w projekcie.

**Wizualizacja:** scatter plot $(\sigma_p, \mu_p)$ z 10 000+ losowych portfeli (Monte Carlo). Górna obwiednia chmury punktów zarysowuje kształt granicy efektywnej.

---

## 4. K-means — klasteryzacja aktywów

*(nawiązanie do [[05 - Elementy Kompresji i K-means]])*

### Cel w projekcie

Spółki reprezentowane jako punkty w przestrzeni $(\mu_i, \sigma_i)$. K-means grupuje je wg podobnego profilu ryzyko-zwrot. Z każdego klastra wybierany jest reprezentant (spółka najbliższa centroidowi), co redukuje liczbę aktywów do $k$ — jednego per klaster.

### Funkcja kosztu (WCSS)

$$
J = \sum_{i=1}^{N} \|x_i - c_{a_i}\|^2
$$

gdzie $a_i = \arg\min_j \|x_i - c_j\|^2$ — przypisanie punktu do najbliższego centroidu.

### Algorytm

1. **Inicjalizacja:** wybierz $k$ centroidów (losowo lub k-means++)
2. **Przypisanie:** każdy punkt → najbliższy centroid
3. **Aktualizacja:** $c_j = \text{mean}(\{x_i : a_i = j\})$
4. **Sprawdź zbieżność:** jeśli centroidy się nie zmieniły — stop; inaczej → krok 2

Zbieżność gwarantowana w skończonej liczbie kroków (J maleje monotonicznie).

### K-means++

Ulepszona inicjalizacja centroidów:

$$
P(x_i \text{ wybrany}) \propto D(x_i)^2
$$

gdzie $D(x_i) = \min_{j < t} \|x_i - c_j\|^2$ — kwadrat dystansu do już wybranych centrów.

Gwarancja: oczekiwany koszt $O(\log k)$ razy gorszy od optimum globalnego.

### Wybór k

- **Metoda łokcia:** wykres $J(k)$ — szukaj "kolana" (punkt gwałtownego wypłaszczenia)
- **Silhouette Score:**

$$
s(i) = \frac{b(i) - a(i)}{\max(a(i),\ b(i))} \in [-1, 1]
$$

gdzie $a(i)$ = średnia odległość do innych punktów w tym samym klastrze,
$b(i)$ = średnia odległość do punktów w najbliższym obcym klastrze.

$s(i) \approx 1$ → punkt dobrze przypisany; $s(i) < 0$ → punkt w złym klastrze.

### Ograniczenia

- Wymaga podania $k$ z góry
- Zakłada kuliste, równoliczne klastry (nie zawsze spełnione w danych giełdowych)
- Wrażliwy na skalę — **obowiązkowy StandardScaler przed uruchomieniem**
- Wynik zależy od inicjalizacji — uruchom wielokrotnie, wybierz najlepsze $J$

---

## 5. Algorytm genetyczny

*(nawiązanie do [[03 - Optymalizacja i Loss Functions]])*

### Reprezentacja (kodowanie chromosomu)

Chromosom to wektor wag portfela $\mathbf{w} \in \mathbb{R}^n$. Po każdej operacji genetycznej stosuje się normalizację:

$$
w_i \leftarrow \frac{\max(w_i, 0)}{\sum_j \max(w_j, 0)}
$$

co gwarantuje $\sum w_i = 1$, $w_i \geq 0$ (ograniczenia twarde).

### Selekcja

**Turniejowa (używana w projekcie):** losuj $k$ osobników, wybierz najlepszego wg Sharpe'a.

$$
\text{winner} = \arg\max_{x \in \text{tournament}} S(x)
$$

Zaleta: działa przy ujemnych wartościach fitness (inaczej niż ruletkowa).

**Ruletkowa (do porównania):**

$$
P(x_i \text{ wybrany}) = \frac{S(x_i)}{\sum_j S(x_j)}
$$

Wymaga $S(x_i) > 0$ — problematyczne gdy portfele mają ujemny Sharpe.

### Crossover (krzyżowanie)

**Convex crossover** — naturalny dla wag ciągłych:

$$
\mathbf{c} = \alpha \cdot \mathbf{p}_1 + (1 - \alpha) \cdot \mathbf{p}_2, \quad \alpha \sim U(0, 1)
$$

Po crossover normalizacja → gwarancja $\sum w_i = 1$, $w_i \geq 0$.

### Mutacja

**Addytywna Gaussowska:**

$$
w_i' = w_i + \varepsilon_i, \quad \varepsilon_i \sim \mathcal{N}(0, \sigma_{\text{mut}}^2)
$$

Następnie: $\text{clip}(0, \infty)$ i normalizacja.

Zbyt duże $\sigma_{\text{mut}}$ → dryf losowy (brak zbieżności).
Zbyt małe $\sigma_{\text{mut}}$ → przedwczesna zbieżność (utknięcie w minimum lokalnym).

### Elitaryzm

Top-$k$ osobników przenoszone bezpośrednio do następnej generacji bez modyfikacji.
Gwarantuje monotoniczność najlepszego fitness w historii.

### Pętla GA — schemat

$$
\text{populacja}_0 \xrightarrow{\text{fitness}} \text{select} \xrightarrow{\text{crossover}} \text{mutate} \xrightarrow{\text{elitism}} \text{populacja}_1 \to \ldots
$$

---

## 6. Monte Carlo w finansach

*(nawiązanie do [[02 - Elementy Losowania i Teoria Prawdopodobienstwa]])*

**Zasada:** estymacja przez wielokrotne losowe próbkowanie.

**W projekcie:** losowe portfele z sympleksu prawdopodobieństwa:

$$
\mathbf{w} \sim \text{Dirichlet}(\mathbf{1}_n) \implies \sum w_i = 1, \ w_i \geq 0
$$

Dla każdego losowego $\mathbf{w}$ obliczamy $(\sigma_p, \mu_p, S)$ i zaznaczamy punkt na wykresie.

**Liczba symulacji:** $10\,000$–$100\,000$ dla dobrego pokrycia przestrzeni.

**Wyniki Monte Carlo:**
- **Portfel MVP** (Minimum Variance Portfolio): $\mathbf{w}^* = \arg\min \sigma_p$
- **Portfel max Sharpe (MC):** $\mathbf{w}^* = \arg\max S(\mathbf{w})$ wśród próbek — baseline dla GA

**Granica efektywna** wyłania się jako górna obwiednia chmury punktów na scatter plot $(\sigma_p, \mu_p)$.

**Dlaczego GA > Monte Carlo:**
Monte Carlo próbkuje przestrzeń losowo — GA kieruje poszukiwanie ku obszarom o wyższym Sharpe'ie. GA powinien znaleźć wyższy Sharpe niż najlepszy z $N$ losowych portfeli.
