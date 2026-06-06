---
tags:
  - mpsi
  - projekt
  - eksperymenty
wiki_type: analysis
last_updated: 2026-05-28
---

# Eksperymenty — Log Hiperparametrów GA

Wyniki z [[portfolio_optimization|notebooka]]. Optymalizujemy **22 reprezentantów** wyłonionych przez k-means
z uniwersum **~186 spółek** S&P 500 (patrz [[Dane]], [[README]]). Każda konfiguracja uruchomiona na
**5 ziarnach** — `Sharpe GA` = średnia, `± std` mierzy stabilność. Metoda **ceteris paribus** (względem E01).

> **Najlepsza konfiguracja: E04** (słaba mutacja) — Sharpe **1.1581** = **97.7%** optimum analitycznego
> (SLSQP = 1.1860). Najgorsza: **E07** (mało elitaryzmu) — 1.1135.

---

## Tabela eksperymentów

| ID | pop_size | n_gen | sigma_mut | cx_rate | n_elites | tourn_k | Sharpe GA (śr.) | Sharpe MC_best | Gen zbieżności | Czas [s] | Uwagi |
|----|----------|-------|-----------|---------|----------|---------|-----------------|----------------|----------------|----------|-------|
| **E04** | 100 | 300 | **0.02** | 0.8 | 5 | 3 | **1.1581 (±0.0042)** | 0.908 | 248 | 0.65 | **NAJLEPSZY i najstabilniejszy** |
| E11 | 100 | **500** | 0.05 | 0.8 | 5 | 3 | 1.1518 (±0.0129) | 0.908 | 345 | 1.07 | dłuższa ewolucja — drugi wynik |
| E08 | 100 | 300 | 0.05 | 0.8 | **10** | 3 | 1.1434 (±0.0095) | 0.908 | 229 | 0.61 | silny elitaryzm — stabilny |
| E10 | 100 | 300 | 0.05 | 0.8 | 5 | **5** | 1.1421 (±0.0047) | 0.908 | 235 | 0.64 | silna selekcja |
| E01 | 100 | 300 | 0.05 | 0.8 | 5 | 3 | 1.1378 (±0.0206) | 0.908 | 230 | 0.64 | baseline (wysoka wariancja) |
| E02 | 50 | 300 | 0.05 | 0.8 | 5 | 3 | 1.1329 (±0.0132) | 0.908 | 168 | 0.31 | mniejsza populacja |
| E06 | 100 | 300 | 0.05 | **0.5** | 5 | 3 | 1.1298 (±0.0130) | 0.908 | 177 | 0.59 | mniej krzyżowania |
| E03 | 200 | 300 | 0.05 | 0.8 | 5 | 3 | 1.1236 (±0.0155) | 0.908 | 168 | 1.31 | większa populacja — szybsza, lecz gorsza |
| E05 | 100 | 300 | **0.10** | 0.8 | 5 | 3 | 1.1184 (±0.0191) | 0.908 | 246 | 0.64 | silna mutacja — dryf losowy |
| E09 | 100 | 300 | 0.05 | 0.8 | 5 | **2** | 1.1144 (±0.0108) | 0.908 | 237 | 0.63 | słaba selekcja |
| E07 | 100 | 300 | 0.05 | 0.8 | **2** | 3 | 1.1135 (±0.0185) | 0.908 | 227 | 0.66 | **NAJGORSZY** — tracone elity |

_Sharpe GA = średnia z 5 ziaren; Sharpe MC_best = stały baseline (najlepszy z 30 000 portfeli Monte Carlo)._

---

## Analiza wrażliwości

### Wpływ sigma_mut *(E04 1.1581 · E01 1.1378 · E05 1.1184)* — **parametr dominujący**

Najsilniejszy efekt. **Słaba mutacja (E04, 0.02)** daje najwyższy i **najstabilniejszy** wynik (std 0.0042 —
najmniejsza w całej tabeli): populacja precyzyjnie eksploatuje okolice optimum. **Silna mutacja (E05, 0.10)**
obniża wynik (1.1184) i mocno zwiększa wariancję (std 0.0191) — to **dryf losowy**: dobre rozwiązania są
rozbijane szybciej, niż selekcja je utrwala.

### Wpływ elitaryzmu *(E07 1.1135 · E01 1.1378 · E08 1.1434)* — **drugi najważniejszy**

W tym trudniejszym (22-wymiarowym) problemie elitaryzm jest kluczowy. **Mało elit (E07, 2)** daje najgorszy
i niestabilny wynik (tracone najlepsze osobniki między generacjami); **dużo elit (E08, 10)** wyraźnie poprawia
i stabilizuje (std 0.0095).

### Wpływ siły selekcji (tourn_k) *(E09 1.1144 · E01 1.1378 · E10 1.1421)*

Silniejsza presja selekcji (E10, k=5) wyraźnie przewyższa słabą (E09, k=2). Większy turniej częściej promuje
dobre osobniki na rodziców — szybsza i lepsza zbieżność, bez oznak przedwczesnej zbieżności.

### Wpływ pop_size *(E02 1.1329 · E01 1.1378 · E03 1.1236)*

Efekt **niejednoznaczny i drugorzędny**. pop=100 (E01) wypadło najlepiej z trójki; pop=200 (E03) zbiegło
najszybciej (168 gen), ale przy stałych 300 generacjach trafiło średnio gorzej — większa populacja nie
gwarantuje lepszego optimum przy ustalonym budżecie generacji.

### Wpływ cx_rate *(E01 1.1378 · E06 1.1298)*

Obniżenie krzyżowania do 0.5 (E06) lekko pogarsza wynik — mutacja sama nie zapewnia tyle eksploracji co
krzyżowanie wypukłe, ale efekt jest umiarkowany.

### Wpływ n_gen *(E01 1.1378 · E11 1.1518)*

Wydłużenie do 500 generacji (E11) zauważalnie poprawia wynik (drugi najlepszy) — w 22-wymiarowym problemie
GA potrzebuje więcej czasu na konwergencję (zbieżność ~345 gen vs ~230 w baseline).

---

## Porównanie GA vs Monte Carlo (+ walidacja SLSQP)

| Metryka | SLSQP (ground truth) | GA strojony (E04, śr.) | GA baseline (E01, śr.) | Monte Carlo (30k) | Równe wagi reprez. |
|---------|----------------------|------------------------|------------------------|-------------------|--------------------|
| Sharpe ratio | **1.1860** | 1.1581 | 1.1378 | 0.9083 | 0.2509 |
| μp (roczny) | 30.9% | 29.6% | 30.0% | 24.4% | 8.8% |
| σp (roczny) | 22.7% | 22.1% | 22.9% | 22.4% | 19.1% |
| Liczba ocen fitness | — (analityczny) | 30 000 | 30 000 | 30 000 | — |
| % optimum | 100% | **97.7%** | 95.9% | **76.6%** | 21.2% |

**Wniosek:** przy **identycznym budżecie 30 000 ocen** GA osiąga ~98% optimum analitycznego, a ślepe
losowanie Monte Carlo tylko ~77%. W 22-wymiarowym sympleksie losowanie nie pokrywa przestrzeni — ukierunkowana
ewolucja jest dużo efektywniejsza. (Naiwne równe wagi 22 reprezentantów dają zaledwie 0.25, bo część
reprezentantów to spółki o wysokim ryzyku — wartość tkwi w *optymalizacji* wag, nie w ich równym rozłożeniu.)

---

## Wnioski końcowe

1. **E04 (słaba mutacja, 0.02)** to najlepsza i najstabilniejsza konfiguracja — strojenie w kierunku
   *eksploatacji* (słaba mutacja + silny elitaryzm + wyraźna selekcja) przewyższa nastawiony na eksplorację baseline.
2. **Strojenie ma duże znaczenie.** Rozpiętość średnich między najlepszą (E04, 1.158) a najgorszą (E07, 1.114)
   konfiguracją to **0.045** — w przeciwieństwie do trywialnego problemu kilku aktywów (gdzie wszystkie warianty
   zbiegają identycznie). To uzasadnia optymalizację 22 reprezentantów zamiast np. 6.
3. **Zbieżność:** baseline plateau ~230 generacji; E11 (500 gen) wciąż poprawia wynik — w wyższym wymiarze
   warto dać GA więcej czasu.
4. **GA konsekwentnie bije MC** (1.16 vs 0.91 przy 30k ocen) i osiąga ~98% analitycznego optimum (SLSQP).
5. **Dryf losowy** wyraźnie widoczny w E05 (mutacja 0.10): niski wynik + najwyższa wariancja.

_Źródło: [[portfolio_optimization]] (notebook), SEED=42, dane 2021-01 → 2024-01, uniwersum ~186 → 22 reprezentantów._
