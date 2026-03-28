# Reinforcement Learning(RL) Trading Agent with PPO   

## 1. What This Project Does

- Builds a trading AI agent that learns from **historical EUR/USD hourly data** using **reinforcement learning**.
- The agent learns by taking trades, receiving rewards for good outcomes, and penalties for bad ones.
- The goal is to develop a policy that improves trading decisions over time, similar to how a human learns by trial and error.

---

## 2. Core Reinforcement Learning Ideas

### Main components

- **Agent**: the model making trading decisions.
- **Environment**: the market simulation where the agent trades.
- **Actions**: decisions such as no trade, long, or short.
- **Reward**: positive or negative feedback based on trade outcome.
- **Policy**: the strategy the agent learns for choosing actions from market states.

### Key idea

- This is a **model-free** reinforcement learning setup.
- The agent does not receive a full market map in advance.
- It learns directly from interaction with historical data through trial and error.

---

## 3. Project Structure

- The project is split into multiple Python files instead of a notebook.
- This makes the code easier to manage, reuse, and test.
- The main files described in the lecture are:
  - `indicators.py`
  - `trading_environment.py`
  - `train_agent.py`
  - `test_agent.py`

---

## 4. Data Used

### Training data
- EUR/USD hourly candles from **2020 to 2023**.

### Testing data
- EUR/USD hourly candles from **2023 to 2025**.
- This is used as **out-of-sample data** to evaluate the trained agent on unseen market conditions.

---

## 5. `indicators.py`: Load and Preprocess Data

### Purpose
- Read raw OHLC data from CSV.
- Clean and sort the data.
- Add technical indicators.
- Remove empty rows after indicator calculation.

### Indicators added
- **RSI 14**
- **Moving average 20**
- **Moving average 50**
- **ATR**
- **20-period moving average slope approximation**

### Notes
- The so-called slope is not a regression slope.
- It is simply the difference between consecutive moving average values.
- This file is the best place to add custom indicators if you want to improve the system.

### Suggested workflow
1. Load the CSV.
2. Sort the index.
3. Compute indicators.
4. Drop NaN rows.
5. Return the final feature dataframe.

---

## 6. `trading_environment.py`: Define the Trading Environment

### Purpose
- Create a custom environment by inheriting from `gym.Env`.
- Let the agent interact with historical candles step by step.

### Main configuration

- **Window size**: default `30`
  - The agent observes the last 30 candles before deciding.
- **Stop-loss options**:
  - 60 pips
  - 90 pips
  - 120 pips
- **Take-profit options**:
  - 60 pips
  - 90 pips
  - 120 pips
- **Trade actions**:
  - `0`: no trade
  - `1`: take a trade
  - Trade direction can be long or short depending on logic in the environment.

### Environment state
- Starting equity: `$10,000`
- Current step: starts at `0`
- Max slippage: `0`
- Positions list: initially empty
- Equity curve history: stored for plotting later

---

## 7. Observation Function

### `get_observation()`
- Returns the last `window_size` candles as a NumPy array.
- Includes all feature columns, not only raw prices.
- Handles the beginning of the dataset carefully when there is not enough history yet.

### Output shape
- `window_size x number_of_features`

### Why it matters
- The agent needs a fixed-size observation before every decision.
- This is the market state that the policy sees.

---

## 8. Reward Function

### Purpose
- Reward the agent when a trade is profitable.
- Penalize the agent when a trade loses money.

### Reward logic
- Positive profit and loss gives a **positive reward**.
- Negative profit and loss gives a **negative reward**.
- Reward is scaled by multiplying PnL by `10,000`.

### Special case
- If a candle hits both stop-loss and take-profit at the same time, the system cannot know which happened first.
- Because the data is candle-based and not tick-based, the implementation takes the safer choice:
  - treat it as a **loss**
  - return a negative reward

### Why this matters
- Reinforcement learning needs a clear and consistent reward signal.
- The reward function is the main learning signal for the agent.

---

## 9. Step and Reset Logic

### `step()`
- Combines:
  - action selection
  - stop-loss/take-profit evaluation
  - reward calculation
  - next observation
  - done flag
  - trade info

### `reset()`
- Resets the environment state for a new episode.
- Restores:
  - current step
  - equity
  - equity curve
  - stored trade history

### `render()`
- Plots the equity curve so performance can be visualized.

---

## 10. Training the Agent

### Model used
- **PPO** from Stable Baselines 3.

### Why PPO
- PPO is a **model-free** RL algorithm.
- It works well for noisy, continuous environments like forex trading.
- It learns directly from trial and error in the environment.

### Training setup
1. Load the 2020–2023 dataset.
2. Build the trading environment.
3. Wrap it in a dummy vectorized environment, as required by Stable Baselines 3.
4. Train the PPO model for **50,000 time steps**.
5. Save the model as a `.zip` file.

### Example training command
```bash
python train_agent.py
```


### What training output shows

- Elapsed time
- Learning rate
- Loss values
- Equity curve growth on training data

---

## 11. Training Behavior

### Observed result

- The equity curve rises on the training data.
- This suggests the agent is learning a useful policy.
- The model seems to understand which actions are being rewarded.


### Important caution

- Good training performance does not guarantee good real-world performance.
- The real test is always out-of-sample evaluation on unseen data.

---

## 12. Testing on Unseen Data

### Testing setup

- Use the saved PPO model from training.
- Load new EUR/USD hourly data from **2023 to 2025**.
- Do not retrain.
- Let the agent trade and build a new equity curve.


### Example command

```bash
python test_agent.py
```


### What happened

- The equity curve was weaker on unseen data than on training data.
- That is not surprising because:
    - the feature set was small,
    - the action space was limited,
    - the training data may have caused some overfitting.

---

## 13. Why Results Can Be Improved

### Limitations in the current version

- Only a few indicators were used:
    - RSI
    - two moving averages
    - ATR
    - a simple moving-average difference
- Stop-loss and take-profit choices were limited to only three values each.
- Training used 50,000 steps, which may overfit the agent.


### Possible improvements

- Add more technical indicators.
- Add custom trading features.
- Use finer stop-loss / take-profit steps.
- Expand the action space.
- Tune PPO hyperparameters.
- Reduce training steps if overfitting appears.
- Try other reward shaping ideas.

---

## 14. Experiment: Reduce Training Steps

### Change made

- Training time steps reduced from **50,000** to **10,000**.


### Effect

- Training still showed positive learning.
- Test equity behavior changed.
- In some cases, a smaller number of steps can reduce overfitting and improve generalization.


### Lesson

- More training is not always better.
- In trading RL, generalization is difficult because market data is noisy.

---

## 15. Practical Interpretation

### What the agent is really learning

- It is not learning “prediction” in the usual supervised sense.
- It is learning a **trading policy**:
    - when to enter,
    - whether to long or short,
    - how to react to stop-loss and take-profit outcomes.


### Why this is difficult

- Trading data is noisy.
- The true signal is mixed with a lot of randomness.
- Reinforcement learning has to find useful patterns in this uncertainty.

---

## 16. README.md Suggested Layout

If you want to turn this into a project README, use this structure:

```markdown
# Reinforcement Learning Forex Trading Agent

## Overview
Short description of the project and objective.

## Features
- PPO-based trading agent
- Custom forex trading environment
- Technical indicator preprocessing
- Training and testing on historical EUR/USD data

## Project Structure
- `indicators.py`
- `trading_environment.py`
- `train_agent.py`
- `test_agent.py`

## Data
- Training data: EUR/USD hourly candles, 2020–2023
- Test data: EUR/USD hourly candles, 2023–2025

## How It Works
1. Load and preprocess data
2. Create custom environment
3. Train PPO agent
4. Test on unseen data

## Reward Design
Explain the PnL-based reward and stop-loss/take-profit logic.

## Training
Explain how to run training and expected output.

## Testing
Explain how to run the test script and inspect the equity curve.

## Possible Improvements
- More indicators
- Better reward shaping
- More action options
- PPO tuning

## Notes
Mention that trading data is noisy and results may overfit.
```


---

## 17. Clear Step-by-Step Workflow

### Step 1: Prepare data

- Load raw EUR/USD candles.
- Add indicators.
- Clean missing rows.


### Step 2: Build the environment

- Define state window.
- Define actions.
- Define reward function.
- Define reset and render behavior.


### Step 3: Train the agent

- Use PPO.
- Run on historical training data.
- Save the model.


### Step 4: Evaluate

- Load unseen data.
- Run the saved agent.
- Plot equity curve.


### Step 5: Improve

- Expand features.
- Tune hyperparameters.
- Adjust action and reward design.

---

## 18. Final Takeaway

- This project shows how reinforcement learning can be used to build a trading agent.
- The main idea is to let the agent learn from historical market interactions and reward feedback.
- Training performance may look good, but testing on unseen data is what really matters.
- The project is a strong starting point for experimenting with RL-based trading systems.

