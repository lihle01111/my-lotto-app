import streamlit as st
import pandas as pd
import itertools
import random

# === LOTTO DATA ===
number_stats = {
    1: (106, 28), 2: (89, 18), 3: (94, 70), 4: (85, 95), 5: (85, 14), 6: (99, 4),
    7: (92, 25), 8: (100, 14), 9: (84, 84), 10: (101, 21), 11: (108, 18),
    12: (91, 11), 13: (104, 18), 14: (109, 32), 15: (82, 14), 16: (96, 4),
    17: (96, 14), 18: (106, 39), 19: (88, 67), 20: (89, 11), 21: (76, 133),
    22: (96, 11), 23: (101, 112), 24: (95, 63), 25: (103, 4), 26: (93, 35),
    27: (94, 7), 28: (99, 46), 29: (109, 7), 30: (89, 4), 31: (88, 28),
    32: (96, 46), 33: (92, 74), 34: (87, 7), 35: (91, 28), 36: (91, 18),
    37: (89, 74), 38: (97, 49), 39: (90, 46), 40: (77, 14), 41: (97, 42),
    42: (101, 60), 43: (93, 56), 44: (93, 21), 45: (105, 21), 46: (107, 18),
    47: (82, 7), 48: (106, 21), 49: (107, 32), 50: (104, 4), 51: (92, 4),
    52: (88, 11)
}

# === LAST 5 DRAWS (Simulated) ===
last_5_draws = [
    {3, 17, 21, 30, 42, 50},
    {9, 22, 24, 39, 44, 48},
    {4, 12, 26, 27, 32, 33},
    {5, 15, 18, 31, 35, 49},
    {7, 10, 14, 25, 34, 46}
]
excluded_numbers = set().union(*last_5_draws)

# === CLASSIFIERS & UTILS ===
def get_frequency_classes():
    freqs = [freq for freq, _ in number_stats.values()]
    sorted_freqs = sorted(freqs)
    third = len(freqs) // 3
    cold_cut = sorted_freqs[third]
    hot_cut = sorted_freqs[-third]

    classes = {}
    for num, (freq, _) in number_stats.items():
        if freq <= cold_cut:
            classes[num] = 'Cold'
        elif freq >= hot_cut:
            classes[num] = 'Hot'
        else:
            classes[num] = 'Lukewarm'
    return classes

def decade(n): return (n - 1) // 10

def passes_wheel_filter(numbers, max_per_decade=4):
    decade_counts = {}
    for n in numbers:
        d = decade(n)
        decade_counts[d] = decade_counts.get(d, 0) + 1
        if decade_counts[d] > max_per_decade:
            return False
    return True

def score_numbers(strategy='hybrid'):
    scored = []
    for num, (freq, delay) in number_stats.items():
        if strategy == 'delay':
            score = delay
        elif strategy == 'frequency':
            score = 100 - freq
        else:
            score = delay * 1.5 + (100 - freq) * 0.5
        scored.append((score, num))
    return sorted(scored, reverse=True)

def generate_prediction_sets(top_n_pool=30, set_size=12, num_sets=10, strategy='hybrid'):
    scored_numbers = score_numbers(strategy)
    top_numbers = [num for _, num in scored_numbers if num not in excluded_numbers][:top_n_pool]

    prediction_sets = []
    attempts = 0
    max_attempts = 1000
    while len(prediction_sets) < num_sets and attempts < max_attempts:
        candidate = sorted(random.sample(top_numbers, set_size))
        if passes_wheel_filter(candidate, max_per_decade=4):
            prediction_sets.append(candidate)
        attempts += 1
    return prediction_sets

def get_deciles(metric='delay'):
    idx = 1 if metric == 'delay' else 0
    sorted_list = sorted(number_stats.items(), key=lambda x: x[1][idx], reverse=True)
    deciles = [sorted_list[i:i+5] for i in range(0, 50, 5)]
    return deciles

def filter_numbers(min_delay=40, max_freq=90):
    return [num for num, (freq, delay) in number_stats.items() if delay >= min_delay and freq <= max_freq]

# === WHEELING FUNCTIONS ===
def is_valid_custom_filter(n, even=None, first_half=None):
    if even is not None:
        if even and n % 2 != 0:
            return False
        if not even and n % 2 == 0:
            return False
    if first_half is not None:
        if first_half and n > 26:
            return False
        if not first_half and n <= 26:
            return False
    return True

def generate_wheeling_sets(
    base_numbers,
    all_numbers=range(1, 53),
    size=6,
    output_count=10,
    filter_even=None,
    filter_first_half=None,
    min_matched=3,
    include_headgame_pairs=None
):
    wheel_sets = []
    used_numbers = [n for n in all_numbers if is_valid_custom_filter(n, filter_even, filter_first_half)]
    
    for combo in itertools.combinations(used_numbers, size):
        match_count = len(set(combo) & set(base_numbers))
        if match_count >= min_matched:
            if include_headgame_pairs:
                if not any((n in include_headgame_pairs) for n in combo):
                    continue
            wheel_sets.append(combo)
            if len(wheel_sets) >= output_count:
                break
    return wheel_sets

# === STREAMLIT APP ===
st.set_page_config(page_title="SA Lotto Dashboard", layout="wide")
st.title("🎰 SA Lotto Predictor & Wheeling System")

# --- Combo Viewer ---
st.header("📊 Combo Viewer")
col1, col2, col3 = st.columns(3)
with col1:
    group_size = st.selectbox("Combo Size", [2, 3, 4], index=2)
with col2:
    score_type = st.selectbox("Score Type", ['combined', 'delay', 'frequency'], index=0)
with col3:
    top_n = st.slider("Top N Results", 5, 50, 10)

def get_top_combos(group_size=4, top_n=10, score_type='combined'):
    combos = []
    for combo in itertools.combinations(number_stats.keys(), group_size):
        delays = [number_stats[n][1] for n in combo]
        freqs = [number_stats[n][0] for n in combo]
        if score_type == 'delay':
            score = sum(delays)
        elif score_type == 'frequency':
            score = sum(freqs)
        else:
            score = sum(d * 1.5 + (100 - f) * 0.5 for d, f in zip(delays, freqs))
        combos.append((score, combo))
    return sorted(combos, reverse=True)[:top_n]

top_combos = get_top_combos(group_size, top_n, score_type)
df_combo = pd.DataFrame([{"Rank": i+1, "Combo": combo, "Score": round(score, 2)} for i, (score, combo) in enumerate(top_combos)])
st.dataframe(df_combo, use_container_width=True)

csv_combo = df_combo.to_csv(index=False).encode("utf-8")
st.download_button("⬇️ Download Combos CSV", csv_combo, f"top_combos_{group_size}numbers.csv", "text/csv")

# --- Decile Breakdown ---
st.header("📈 Decile Breakdown")
metric = st.radio("Sort deciles by", ['delay', 'frequency'], horizontal=True)
deciles = get_deciles(metric)
for i, dec in enumerate(deciles, 1):
    nums = [f"{n} ({number_stats[n][0]}f/{number_stats[n][1]}d)" for n, _ in dec]
    st.write(f"Decile {i}: {', '.join(nums)}")

# --- Delay + Frequency Filter ---
st.header("🔥 Overdue & Underused Filter")
min_delay = st.slider("Minimum Delay", 0, 150, 40)
max_freq = st.slider("Maximum Frequency", 70, 110, 90)
filtered_nums = filter_numbers(min_delay, max_freq)
st.success("Filtered Numbers:\n" + ", ".join(map(str, filtered_nums)))

# --- Prediction Sets ---
st.header("🔮 Predict 10 System Play Sets (12 Numbers)")
pred_strategy = st.radio("Prediction Strategy", ['hybrid', 'delay', 'frequency'], horizontal=True)
freq_classes = get_frequency_classes()
prediction_sets = generate_prediction_sets(strategy=pred_strategy)
labeled_sets = [[f"{n} ({freq_classes[n][0]})" for n in row] for row in prediction_sets]
df_pred = pd.DataFrame(labeled_sets, columns=[f"N{i+1}" for i in range(12)])
st.dataframe(df_pred, use_container_width=True)
csv_pred = pd.DataFrame(prediction_sets).to_csv(index=False).encode("utf-8")
st.download_button("⬇️ Download Prediction Sets CSV", csv_pred, f"lotto_prediction_sets_{pred_strategy}.csv", "text/csv")

# --- Wheeling System ---
st.header("🔁 Wheeling System Generator")
st.markdown("Based on draw of **6 August 2025**: `49, 26, 3, 18, 31, 5`")
base_draw = [49, 26, 3, 18, 31, 5]
headgame_21 = [6, 38, 42, 52, 5]
headgame_18 = [20, 32, 31, 24, 2, 6]
all_headgame = headgame_21 + headgame_18
col1, col2, col3 = st.columns(3)
with col1:
    even_filter = st.selectbox("Even Filter", ["Any", "Only Even", "Only Odd"])
    even = None if even_filter == "Any" else (True if even_filter == "Only Even" else False)
with col2:
    half_filter = st.selectbox("Half Filter", ["Any", "1–26", "27–52"])
    first_half = None if half_filter == "Any" else (True if half_filter == "1–26" else False)
with col3:
    min_match = st.slider("Minimum Match With Base", 1, 6, 3)

wheel_sets = generate_wheeling_sets(
    base_numbers=base_draw,
    filter_even=even,
    filter_first_half=first_half,
    min_matched=min_match,
    include_headgame_pairs=all_headgame
)

if wheel_sets:
    st.success(f"Generated {len(wheel_sets)} valid wheeled sets.")
    df_wheel = pd.DataFrame(wheel_sets, columns=[f"N{i+1}" for i in range(6)])
    st.dataframe(df_wheel, use_container_width=True)
    csv_wheel = df_wheel.to_csv(index=False).encode("utf-8")
    st.download_button("⬇️ Download Wheeled Sets CSV", csv_wheel, "wheeled_combos.csv", "text/csv")
else:
    st.warning("No valid sets found with current filter. Try adjusting filters or min match.")
