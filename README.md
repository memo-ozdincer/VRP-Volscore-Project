This project implements a simplified version of Barclays' volatility trading strategy that aims to outperform traditional equity exposure by short vol strategies using the Volatility Risk Premium (VRP) in options markets.

https://amarketplaceofideas.com/wp-content/uploads/2021/08/Barclays_US_Equity_Derivatives_Strategy_Impact_of_Retail_Options_Trading.pdf

The fundamental idea was that higher than expected far-OTM options activity could initially cause a discrepancy in IV and RV. However, their "volscore" model was more concerned with the downstream effects from this, caused by firms using covered calls to supply the short-expiry far-OTM calls to the retail market. Essentially, far-OTM retail activity is easy to account for in the market; however, the huge amount of additional exposure to the underlying security taken on by hedge funds selling these far-OTM covered calls was unexpected, especially with such short expiration dates. This, in theory causes unexpected movements in options markets (even supposedly in the LEAPs, as I guess a lot of people were selling "poor mamn's cobvered calls"). So, all of this caused inefficiencies in options deltas that were just great enough for Barclays and other trading firms to take advantage. In their paper, they say their proprietary model could account for it, and I was really interested to see how this could be done, which led me to this implementation.

So far, I implemented the additional increased retail activity but not the downstream effects, and thought of doing so with deep learning in the future.

1) Implementation

#### 1.1. Introduction

This repository contains a simplified end-to-end system to experiment with the “Volatility Risk Premium” (VRP) and how it might be integrated into an equity-centric portfolio, as outlined here (https://indices.cib.barclays/dms/Public%20marketing/Volatility_Risk_Premium.pdf). The project is primarily composed of:
• A C++-based “VolScore” library (wrapped in Python via pybind11) to compute the given security's realized volatility metrics.  
• A Python-based data pipeline and VRP calculations.  
• A machine learning hook written for TensorFlow, where I plan to add a VRP "correction" model.
• A simple backtester based on the second paper cited above, which shows how a short vol strategy can be implemented into a portfolio rather than equity exposures, including performance.

A lot of research by Hedge funds have been put forward that VRP can serve as a return source, sometimes outperforming equity market exposure under certain stress events (high concentration of retail investors). It references their internal "volscore" algorithm, and my code is a recreation of what I believe that could be. So far it does the following
• Computing realized volatility.  
• Computing implied volatility (e.g., from a proxy like VIX).  
• Forming the difference as VRP.  
• Optionally using a machine learning model to adjust the basic VRP estimates.  
• Replacing portions of an equity-based portfolio with short vol exposure in a naive backtest.

#### 1.2. Prerequisites

• C++17 compiler (e.g., g++ or clang++)  
• CMake >= 3.10  
• Python 3.9+  
• Pybind11 >= 2.10  
• (Optional) yfinance for real market data  
• (Optional) TensorFlow for the ML portion  

#### 1.3. Building and Installing the C++ “VolScore” Component

1. Navigate to the cpp directory:  
   ```
   cd volscore_vrp_project/cpp
   ```
2. Create and enter a build folder:  
   ```
   mkdir build && cd build
   ```
3. Run CMake and build:  
   ```
   cmake ..
   make
   ```
4. (Optional) Run the C++ test:  
   ```
   ctest
   ```
   This will run the simple test_volscore.cpp unit test for the VolScore library.

#### 1.4. Python Pybind11 Wrapper

In the “python/volscore_wrapper” folder, we have a pybind11-based wrapper that exposes the C++ VolScore functionalities for Python usage.

1. Install Python dependencies (in a virtual environment or otherwise) by running:  
   ```
   pip install -r requirements.txt
   ```
2. Build and install the wrapper in-place:  
   ```
   cd volscore_vrp_project/python
   python setup.py build_ext --inplace
   ```
3. Verify you can now do:
   ```python
   import volscore_wrapper
   vs = volscore_wrapper.VolScore()
   vs_val = vs.computeVolScore([100, 101, 102, 99])
   print(vs_val)
   ```

#### 1.5. Running a Naive VRP Computation

Inside `vrp_computation.py`, we define a class that:
• Uses the C++ `VolScore` object to compute realized vol.  
• Fetches (or mocks) implied vol data.  
• Calculates a naive “VRP = ImpliedVol - RealizedVol.”

Simply run:
```
python vrp_computation.py
```
It will print out the computed VRP for a small set of data points (or actual pulled data via yfinance).

#### 1.6. Machine Learning Adjustments

We have a basic “ml_placeholder.py” and “adjust_vrp_ml.py” that show how we might incorporate a TensorFlow model to “predict” or “adjust” volatility. The code can:
• Load or define a trivial neural network.  
• Provide a small function “adjust_vrp_with_ml(vrp_value)” to shift the naive VRP by the model’s inference.  

To check it:
```
python ml_placeholder.py
```
This will display a summary of a dummy Keras model. Actual training pipelines or time-series models (e.g., LSTMs) can be substituted in place of these placeholders.

#### 1.7. Minimal Backtesting

`simple_backtester.py` sets up a basic event-driven loop, day by day (or row by row), to:
• Compute the VRP.  
• Decide on a “short volatility” position if VRP > some threshold. (Alternatively, one might do “equity vs. short vol” in the same loop.)  
• Track daily PnL in a list.  

To run:
```
python simple_backtester.py
```
Check the console for final PnL or any rolling performance metrics.

---

### 2) Theory

#### 2.1. Background

• The VRP is the difference between implied volatility (often measured by an index like the VIX) and realized volatility. Historically, implied volatility tends to exceed subsequent realized volatility, leading to a so-called “premium.”  
• This premium can be captured via short volatility strategies, e.g., short variance swaps or short ATM options.  
• While VRP strategies often exhibit some positive correlation with equities (especially in crises), there are historical scenarios (e.g., burst of tech bubble or prolonged markets with persistently higher implied vol) in which VRP exposures can deliver strong risk-adjusted returns.  
• The paper also discusses practical sizing in a multi-asset portfolio. Replacing a part of one’s equity allocation with short vol exposure can (depending on implementation) increase diversification benefits, reduce tail risks, and possibly boost Sharpe ratios.

#### 2.2. Math

Let’s say we track a daily closing price series S(t). Then:

1) Realized Vol:  
   RVol = std( (S(t+1) - S(t)) / S(t) ), over T days  
2) Implied Vol:  
   Typically observed from an option chain or an implied volatility index (like the VIX).

Hence VRP = ImpliedVol - RealizedVol (shown in volatility points).  
A short volatility strategy attempts to, on average, pocket the implied minus realized difference.  

#### 2.3. Machine Learning Possible usage

We can refine VRP forecasts. For example, let x(t) be a feature vector capturing phenomenon such as:
• Momentum or macro environment (e.g., market “fear”).  
• Historical realized vol and prior implied vol terms.  
• Other exogenous signals (yield curve slopes, credit spreads, etc.).  

I started this project thinking I would train an LSTM or feed-forward model y = f(x) to predict future realized volatility or directly predict VRP. Then the strategy might short volatility more aggressively when the predicted VRP is high, or hold less exposure when the predicted VRP is low. This approach merges VRP with data-driven modeling to better handle regime shifts. However, I lost interest in the project after spending my winter break on it.

#### 2.4. Potential Limitations

• Path dependency: Some short vol instruments have path-dependent risk (like variance swaps).  
• Sudden volatility spikes: Large drawdowns can occur if realized volatility surges far beyond implied levels.  
• Data alignment and realistic transaction costs: Real implementations must account for cost, liquidity constraints, etc.

---

## Instructions to Run:

1) Clone or download this repo.  
2) Build the C++ library as described (CMake + make).  
3) Install the Python wrapper (setup.py build_ext --inplace).  
4) Install the Python dependencies:  
   ```
   pip install -r requirements.txt
   ```
5) Run each script in “python/” stand-alone or via your chosen workflow, e.g.:  
   ```
   python vrp_computation.py
   python simple_backtester.py
   ```
   Possibly modify parameters to see how different thresholds or volumes of short vol exposure might behave.
