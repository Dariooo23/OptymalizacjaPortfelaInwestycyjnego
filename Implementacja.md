# Implementacja — Pseudokod i Decyzje

Każda sekcja = jeden krok planu pracy. Każda zawiera pseudokod, kluczowe
decyzje projektowe, pułapki i miejsce na własne notatki.

---

## Krok 1: Pobranie i normalizacja danych

### Pseudokod

```
download(tickers, start, end, auto_adjust=True)
prices = adj_close.dropna(axis=1, how="any")
returns = log(prices / prices.shift(1)).dropna()
mu_vec = returns.mean() * 252
cov_mat = returns.cov() * 252
```

### Kluczowe decyzje

- **Okres danych:** 2021-01-01 → 2024-01-01 (3 lata, uzasadnienie: zob. [[Dane]])
- **Stopa wolna od ryzyka:** `RF = 0.04` (4% — US T-bills 2024)
- **Obsługa braków:** `dropna(axis=1, how="any")` — usuń spółki z jakimikolwiek brakami

### Pułapki

- `auto_adjust=True` obowiązkowe — inaczej ceny nie uwzględniają dywidend
- yfinance ≥0.2 może zwracać MultiIndex — sprawdź `raw["Close"]` vs `raw.xs("Close", level=1, axis=1)`
- Nie mieszaj skorygowanych i nieskorygowanych cen w jednej analizie

### Notatki

> *(uzupełnij podczas implementacji)*

---

## Krok 2: K-means od zera

### Pseudokod

```python
def kmeans_plus_plus_init(X, k):
    centroids = [X[random_index]]
    for _ in range(k - 1):
        D_sq = [min(dist(x, c)**2 for c in centroids) for x in X]
        probabilities = D_sq / sum(D_sq)
        next_centroid = X[sample(probabilities)]
        centroids.append(next_centroid)
    return centroids

def kmeans(X, k, max_iters=300, tol=1e-6):
    centroids = kmeans_plus_plus_init(X, k)
    for _ in range(max_iters):
        # Przypisanie
        labels = [argmin(dist(x, c) for c in centroids) for x in X]
        # Aktualizacja
        new_centroids = [mean(X[labels == j]) for j in range(k)]
        # Zbieżność
        if max(dist(c, nc) for c, nc in zip(centroids, new_centroids)) < tol:
            break
        centroids = new_centroids
    return centroids, labels
```

### Przestrzeń cech

Wejście do k-means: macierz $N \times 2$ gdzie kolumny to $(\mu_i, \sigma_i)$ — annualizowane, przed StandardScalerem.

```python
from sklearn.preprocessing import StandardScaler  # lub implementacja własna

features = np.column_stack([mu_vec, sigma_vec])    # (n_assets, 2)
scaler_mean = features.mean(axis=0)
scaler_std = features.std(axis=0)
X = (features - scaler_mean) / scaler_std          # StandardScaler from scratch
```

### Wybór k

Testuj $k \in \{3, 4, 5, 6\}$:

```
for k in range(3, 7):
    centroids, labels = kmeans(X, k)
    inertia = sum(dist(x, centroids[label])**2 for x, label in zip(X, labels))
    silhouette = silhouette_score(X, labels)
    print(k, inertia, silhouette)
```

Wybierz k wg metody łokcia lub max silhouette.

### Wyłonienie reprezentantów klastrów

```python
representatives = []
for j in range(k):
    cluster_points = [i for i, l in enumerate(labels) if l == j]
    rep_idx = min(cluster_points, key=lambda i: dist(X[i], centroids[j]))
    representatives.append(tickers[rep_idx])
```

### Kluczowe decyzje

- **Normalizacja:** obowiązkowy StandardScaler — bez niego σ (zakres ~0.15–0.60) zdominuje μ (zakres ~-0.1–0.5), k-means klasteryzuje tylko po σ
- **Wielokrotne uruchomienie:** k-means lokalnie optymalny — uruchom 10× z różną inicjalizacją, wybierz min WCSS
- **Implementacja od zera:** nie używaj `sklearn.cluster.KMeans` — projekt wymaga implementacji własnej

### Pułapki

- Klaster z jednym punktem — centroid = ten punkt, silhouette niezdefiniowane; dodaj obsługę
- Pusta inicjalizacja — jeśli k-means++ wybierze identyczny punkt dwa razy, zabezpiecz `set`
- Nie porównuj WCSS między różnymi k — maleje monotonicznie z k, nie używaj do wyboru k bez normalizacji

### Notatki

> *(uzupełnij podczas implementacji)*

---

## Krok 3: Symulacja Monte Carlo

### Pseudokod

```python
N_SIMULATIONS = 10_000
RF = 0.04
n_assets = len(representatives)
mu_rep = mu_vec[representatives]
cov_rep = cov_mat.loc[representatives, representatives].values

results = []
for _ in range(N_SIMULATIONS):
    # Losowy portfel z sympleksu prawdopodobieństwa
    w = np.random.dirichlet(np.ones(n_assets))
    mu_p  = w @ mu_rep
    var_p = w @ cov_rep @ w
    sigma_p = np.sqrt(var_p)
    sharpe  = (mu_p - RF) / sigma_p
    results.append((sigma_p, mu_p, sharpe, w))

results = np.array([(s, m, sh) for s, m, sh, _ in results])
best_mc_idx = results[:, 2].argmax()
```

### Kluczowe decyzje

- **Dirichlet(1):** równomierny rozkład na sympleksie — poprawne losowanie portfeli long-only. **Nie** używać `np.random.uniform` + normalizacja (daje niespójny rozkład).
- **Zakres:** testuj N = 10 000 dla szybkości, 100 000 dla publikacji
- **Reprezentanci klastrów jako aktywa:** MC i GA działają na podzbiorze z k-means

### Pułapki

- `np.sqrt(w @ cov @ w)` może zwrócić NaN jeśli wynik ujemny (macierz nieokreślona) — dodaj `assert np.all(np.linalg.eigvals(cov_rep) >= -1e-10)`
- Kowariancja musi być z tych samych spółek co $\mu$ — indeksuj konsekwentnie
- Wynik MC to **baseline** do porównania z GA w raporcie

### Notatki

> *(uzupełnij podczas implementacji)*

---

## Krok 4: Algorytm genetyczny

### Pseudokod

```python
def initialize_population(pop_size, n_assets):
    return np.random.dirichlet(np.ones(n_assets), size=pop_size)

def fitness(population, mu, cov, rf):
    mu_p = population @ mu
    var_p = np.einsum('ij,jk,ik->i', population, cov, population)
    sigma_p = np.sqrt(var_p)
    return (mu_p - rf) / sigma_p

def tournament_select(population, fit, k=3):
    idx = np.random.choice(len(population), size=k, replace=False)
    return population[idx[np.argmax(fit[idx])]]

def crossover(p1, p2):
    alpha = np.random.random()
    child = alpha * p1 + (1 - alpha) * p2
    child = np.clip(child, 0, None)
    return child / child.sum()

def mutate(w, sigma_mut=0.05):
    w = w + np.random.normal(0, sigma_mut, len(w))
    w = np.clip(w, 0, None)
    return w / w.sum()

def genetic_algorithm(mu, cov, rf=0.04,
                      pop_size=100, n_gen=300,
                      n_elites=5, cx_rate=0.8,
                      tourn_k=3, sigma_mut=0.05):
    n_assets = len(mu)
    pop = initialize_population(pop_size, n_assets)
    history = []

    for gen in range(n_gen):
        fit = fitness(pop, mu, cov, rf)
        history.append(fit.max())

        # Elitaryzm
        elite_idx = np.argsort(fit)[-n_elites:]
        elites = pop[elite_idx]

        # Nowa populacja
        new_pop = list(elites)
        while len(new_pop) < pop_size:
            p1 = tournament_select(pop, fit, tourn_k)
            p2 = tournament_select(pop, fit, tourn_k)
            if np.random.random() < cx_rate:
                child = crossover(p1, p2)
            else:
                child = p1.copy()
            child = mutate(child, sigma_mut)
            new_pop.append(child)

        pop = np.array(new_pop)

    fit = fitness(pop, mu, cov, rf)
    best_w = pop[fit.argmax()]
    return best_w, history
```

### Parametry do tunowania

| Parametr | Sugerowany zakres | Wpływ |
|----------|-------------------|-------|
| `pop_size` | 50–500 | większa = wolniej, lepiej eksploruje |
| `n_gen` | 100–500 | więcej = dłuższa zbieżność |
| `sigma_mut` | 0.01–0.15 | za duże = losowy dryf |
| `cx_rate` | 0.6–0.9 | proporcja crossover vs kopia |
| `n_elites` | 2–10 | zapobiega regresji |
| `tourn_k` | 2–5 | wyższe = silniejsza selekcja |

### Kluczowe decyzje

- **Selekcja turniejowa** zamiast ruletkowej — odporna na ujemne Sharpe (lepsza dla cold start)
- **Normalizacja po każdej operacji** — jedyna metoda wymuszenia ograniczeń twardych bez funkcji kary
- **Elitaryzm** — gwarantuje monotoniczność `max(fit)` w historii generacji

### Pułapki

- **Negatywny Sharpe przy selekcji ruletkowej:** jeśli wszystkie portfele mają $S < 0$, ruletkowa nie działa. Używaj turniejowej lub przesuń fitness: `fit_shifted = fit - fit.min() + 1`
- **Brak normalizacji po mutacji:** wagi ujemne lub $\sum \neq 1$ powodują błędne obliczenie Sharpe'a
- **Zbyt krótka ewolucja:** sprawdź krzywą zbieżności — jeśli nie wypłaszczyła się do gen 300, zwiększ n_gen
- **einsum do wariancji:** `np.einsum('ij,jk,ik->i', pop, cov, pop)` liczy $\mathbf{w}^\top \Sigma \mathbf{w}$ dla całej populacji naraz (wektoryzacja)

### Notatki

> *(uzupełnij podczas implementacji)*

---

## Krok 5: Wizualizacje

### Plot 1 — Scatter Monte Carlo z portfelem GA

```python
fig, ax = plt.subplots(figsize=(10, 7))
sc = ax.scatter(mc_sigma, mc_mu, c=mc_sharpe, cmap='viridis',
                alpha=0.4, s=5, label='Monte Carlo')
plt.colorbar(sc, ax=ax, label='Sharpe ratio')
ax.scatter(*ga_point, color='red', s=200, marker='*',
           zorder=5, label=f'GA optimal (S={ga_sharpe:.3f})')
ax.scatter(*mc_best_point, color='orange', s=100, marker='^',
           zorder=5, label=f'MC best (S={mc_best_sharpe:.3f})')
ax.set_xlabel('Roczne ryzyko σ')
ax.set_ylabel('Oczekiwany zwrot μ')
ax.set_title('Granica efektywna — Monte Carlo vs GA')
ax.legend()
```

### Plot 2 — Krzywa zbieżności Sharpe'a

```python
fig, ax = plt.subplots()
ax.plot(history, label='Max Sharpe (generacja)')
ax.axhline(mc_best_sharpe, color='orange', linestyle='--', label='MC best')
ax.set_xlabel('Generacja')
ax.set_ylabel('Sharpe ratio')
ax.set_title('Zbieżność GA')
ax.legend()
```

### Plot 3 — Alokacja portfela GA

```python
fig, ax = plt.subplots()
ax.bar(representatives, best_w)
ax.set_xlabel('Spółka')
ax.set_ylabel('Waga')
ax.set_title('Portfel optymalny GA')
plt.xticks(rotation=45)
```

### Kluczowe decyzje

- Zaznacz zarówno punkt GA jak i MC_best — ułatwia porównanie w raporcie
- Użyj `cmap='viridis'` dla kolorowania wg Sharpe'a — intuicyjne (żółty = wyższy)

### Notatki

> *(uzupełnij podczas implementacji)*
