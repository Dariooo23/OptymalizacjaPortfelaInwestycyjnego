# Dane — Źródła i Normalizacja

Opis danych wejściowych projektu, wyboru spółek, normalizacji i potencjalnych problemów.

---

## Pobieranie danych — yfinance

```python
import yfinance as yf
import numpy as np
import pandas as pd

# Uniwersum: ~186 dużych spółek S&P 500 z 11 sektorów GICS, budowane ze słownika
# SECTOR_GROUPS (sektor -> lista tickerów) w notebooku. Duże uniwersum jest CELOWE:
# pozwala k-means realnie zredukować wymiarowość (186 -> ~22 reprezentantów).
TICKERS = sorted({t for grp in SECTOR_GROUPS.values() for t in grp})  # ~186 tickerów

# Pobierz ceny zamknięcia (automatycznie korygowane o dywidendy i splity)
raw = yf.download(TICKERS, start="2021-01-01", end="2024-01-01",
                  auto_adjust=True, progress=False)
prices = raw["Close"].dropna(axis=1, how="any")  # usuń spółki z brakami (np. MMC)
```

**Uwaga:** `auto_adjust=True` koryguje ceny o dywidendy i splity stock — używaj zawsze.
Nowe wersje yfinance (≥0.2.x) mogą zmienić nazwy kolumn — weryfikuj `raw.columns`.

---

## Wybór spółek — ~186 z S&P 500 (11 sektorów GICS)

Uniwersum celowo **duże** (~186 spółek), aby grupowanie k-means pełniło realną rolę — **redukcję
wymiarowości**. Bezpośrednia optymalizacja 186 wag jest źle uwarunkowana (silnie skorelowane spółki
sektorowe → macierz kowariancji bliska osobliwej, przestrzeń zbyt duża dla przeszukania). k-means grupuje
spółki wg profilu $(\mu,\sigma)$ i wyłania **reprezentantów** (po jednym na klaster) — i **tylko ich wagi**
optymalizujemy algorytmem genetycznym.

Spółek na sektor (po `dropna`): Information Technology 26, Financials 23, Health Care 22, Industrials 21,
Consumer Discretionary 20, Consumer Staples 18, Energy 13, Communication Services 12, Utilities 11,
Real Estate 11, Materials 9 (≈186 łącznie; pełna lista w `SECTOR_GROUPS` w notebooku).

**Efekt redukcji (wynik):** $k=22$ (maksimum średniego silhouette) → **22 reprezentantów**. Optymalizacja na
22 słabo skorelowanych aktywach jest dobrze uwarunkowana i nietrywialna: algorytm genetyczny osiąga ~98%
optimum analitycznego, a losowy Monte Carlo przy tym samym budżecie tylko ~77%. Patrz [[Eksperymenty]].

---

## Normalizacja — logarytmiczne stopy zwrotu

### Dlaczego log-returns, nie zwykłe zwroty

Zwykła stopa zwrotu: $r_t = (P_t - P_{t-1}) / P_{t-1}$

Logarytmiczna stopa zwrotu: $r_t = \ln(P_t / P_{t-1})$

Zalety log-returns:
- **Addytywność w czasie:** $\ln(P_T/P_0) = \sum_{t=1}^T r_t$
- **Symetria:** +100% i -50% dają ten sam bezwzględny wynik (nie tak dla arytmetycznych)
- **Zbliżone do normalności** — kluczowe założenie modelu Markowitza
- **Numeryczna stabilność** dla dużych przedziałów czasu

### Kod normalizacji

```python
# Log-returns
returns = np.log(prices / prices.shift(1)).dropna()

# Annualizacja (252 dni giełdowe w roku)
RF = 0.04  # stopa wolna od ryzyka (4% ~ US T-bills 2024)
MU = 252

mu_vec = returns.mean() * MU          # wektor oczekiwanych zwrotów (roczny)
cov_mat = returns.cov() * MU          # macierz kowariancji (roczna)
sigma_vec = returns.std() * np.sqrt(MU)  # odch. stand. (roczne)
```

### Annualizacja

252 to konwencjonalna liczba dni giełdowych w roku USA.
- `mean * 252` → oczekiwany zwrot roczny (z prawa wielkich liczb)
- `cov * 252` → macierz kowariancji roczna (kowariancja skaluje się liniowo)
- `std * sqrt(252)` → odch. stand. roczne (odch. stand. skaluje się przez `sqrt(T)`)

---

## Potencjalne problemy z danymi

### 1. Survivorship bias

Wybieramy spółki, które *przeżyły* do dzisiaj i są w S&P 500. Spółki, które zbankrutowały lub zostały usunięte z indeksu, nie są brane pod uwagę. **Skutek:** przeceniamy historyczne zwroty całego rynku.

*Ograniczenie modelu — należy wspomnieć w sekcji "Ograniczenia" raportu.*

### 2. Look-ahead bias

Nie wolno używać danych przyszłych przy podejmowaniu "historycznych" decyzji. W projekcie ryzyko jest minimalne (nie robimy backtesting), ale przy rozszerzeniu na rolling window — uwaga!

### 3. Non-stationarity

$\mu$ i $\sigma$ aktywów zmieniają się w czasie (kryzysy finansowe, zmiany sektorowe). Model Markowitza zakłada stacjonarność. **Skutek:** portfel optymalny historycznie może nie być optymalny w przyszłości.

### 4. Braki danych

Jeśli spółka nie ma notowań przez cały wybrany okres (np. IPO w trakcie):

```python
# Opcja 1: usuń spółki z brakami (bezpieczna)
prices = raw["Close"].dropna(axis=1, how="any")

# Opcja 2: forward-fill (wypełnij ostatnią ceną)
prices = raw["Close"].ffill().dropna()
```

Wybrać opcję 1 i udokumentować, które spółki zostały usunięte.

### 5. Wybór okresu danych

| Okres | Zaleta | Wada |
|-------|--------|------|
| 1 rok | mało danych historycznych | świeże, aktualne |
| 3 lata | dobry balans | obejmuje COVID-19 |
| 5 lat | dużo danych | miesza różne reżimy rynkowe |

**Sugerowany wybór w projekcie: 3 lata** (2021-01-01 → 2024-01-01).
Obejmuje okres post-COVID i wzrostu stóp — zróżnicowane warunki rynkowe.
