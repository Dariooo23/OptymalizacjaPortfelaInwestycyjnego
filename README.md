---
tags:
  - mpsi
  - projekt
  - portfolio-optimization
wiki_type: hub
last_updated: 2026-05-28
---

# Projekt MPSI — Optymalizacja Portfela Inwestycyjnego

Projekt semestralny na kurs MPSI (Matematyczne Podstawy Sztucznej Inteligencji).
Cel: budowa optymalnego portfela inwestycyjnego przez klasteryzację aktywów
algorytmem k-means, a następnie wyznaczenie wag portfela maksymalizujących
wskaźnik Sharpe'a przy użyciu algorytmu genetycznego.

**Deliverable:** jeden plik `portfolio_optimization.ipynb` będący jednocześnie
kodem i raportem

---

## Plan pracy

- [x] **Krok 1** — Pobranie i normalizacja danych (yfinance, ~186 spółek S&P 500)
- [x] **Krok 2** — Implementacja k-means od zera — klasteryzacja spółek wg (σ, μ); wyłonienie reprezentantów klastrów
- [x] **Krok 3** — Symulacja Monte Carlo — losowe portfele w przestrzeni ryzyko-zwrot, granica efektywna
- [x] **Krok 4** — Implementacja algorytmu genetycznego od zera (kodowanie chromosomu, selekcja, crossover, mutacja, funkcja celu: Sharpe'a)
- [x] **Krok 5** — Wizualizacje: scatter MC z punktem GA, krzywa zbieżności Sharpe'a po generacjach
- [x] **Krok 6** — Raport: analiza hiperparametrów GA, porównanie z losowaniem, ograniczenia modelu

---

## Status

> **Ukończony** ✅ — pełny pipeline zaimplementowany i wykonany w `portfolio_optimization.ipynb`

## Wyniki (uruchomienie SEED=42, dane 2021-01 → 2024-01)

k-means redukuje **186 → 22 reprezentantów**; optymalizujemy **tylko reprezentantów** (Sharpe = śr. z 5 ziaren):

| Metoda | Sharpe | % optimum |
|--------|--------|-----------|
| SLSQP (ground truth, walidacja) | **1.1860** | 100% |
| **GA strojony (E04, słaba mutacja)** | **1.1581** | **97.7%** |
| GA baseline (E01) | 1.1378 | 95.9% |
| Monte Carlo (30k, ten sam budżet) | 0.9083 | 76.6% |
| Równe wagi reprezentantów | 0.2509 | 21.2% |
| Równe wagi całego uniwersum (1/N, 186) | 0.4340 | 36.6% |

- **Dane:** ~186 spółek S&P 500 z 11 sektorów, 752 dni giełdowych (rzeczywiste, yfinance → cache `prices.csv`).
- **Klasteryzacja:** k=22 (maksimum średniego silhouette); k-means redukuje 186 spółek do 22 reprezentantów.
- **Portfel optymalny (GA):** AVGO ~37%, energia (WMB/EOG/OXY) ~40%, IBM/ORCL ~18% — zwycięzcy 2021–2024.
- **Główny wniosek:** GA (od zera) trafia w ~98% optimum analitycznego; Monte Carlo przy tym samym budżecie tylko ~77%.

> **Decyzja projektowa (krytyka założeń):** uniwersum celowo **duże (~186 spółek)**, aby k-means realnie
> redukował wymiarowość (186 → 22 reprezentantów), a optymalizacji poddajemy **wyłącznie reprezentantów**.
> Na małym uniwersum (np. 20 spółek → 6 reprezentantów) optymalizacja jest trywialna (wszystkie warianty GA
> zbiegają identycznie); dopiero ~22 reprezentantów daje nietrywialny, dobrze uwarunkowany problem, w którym
> GA wyraźnie bije Monte Carlo (98% vs 77% optimum). Szczegóły: sekcja 1.3 notebooka; wyniki: [[Eksperymenty]].

---

## Technologie

| Rola | Narzędzie |
|------|-----------|
| Środowisko | Python 3.12, Jupyter Notebook |
| Dane | yfinance, pandas |
| Obliczenia | NumPy |
| Wizualizacje | matplotlib |
| Walidacja | scipy.optimize (SLSQP) — tylko jako *ground truth*, nie do implementacji algorytmów |
| Wbudowane | brak zewnętrznych bibliotek ML — k-means i GA implementowane od zera |

---

## Nawigacja — strony projektu

- [[Teoria]] — podkład matematyczny (Sharpe, Markowitz, k-means, GA, Monte Carlo)
- [[Dane]] — yfinance, wybór spółek, normalizacja, potencjalne problemy
- [[Implementacja]] — pseudokod każdego modułu, decyzje projektowe, pułapki
- [[Eksperymenty]] — tabela logów eksperymentów z hiperparametrami GA
- [[Raport_szkic]] — gotowa struktura sekcji do przeklejenia do Notebooka

---
