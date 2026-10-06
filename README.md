# Dashboard_Bluepyton_Nikade
Codice Pyton per graficare i report/backtest di Nikade fondo di Trading


import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import matplotlib.patches as mpatches
import matplotlib.gridspec as gridspec
import numpy as np
import glob
import os
from datetime import datetime

# ━━━━━━━━━━━━━━━━━━━━━━━━━━
#  PALETTE  ─  Dark navy + Electric Blue/Cyan  (leggibile, professionale)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━
BG_FIGURE      = '#040d21'
BG_AXES        = '#071a3e'
BG_DONUT       = '#071a3e'
GRID_COLOR     = '#0d2860'
SPINE_COLOR    = '#163580'
TICK_COLOR     = '#ffffff'
LABEL_COLOR    = '#e0f0ff'
TITLE_COLOR    = '#00cfff'
SUBTITLE_COLOR = '#7ec8e3'
TEXT_WHITE     = '#ffffff'

BLUE_LINE      = '#2196f3'
BLUE_FILL      = '#0a3d8f'
CYAN_LINE      = '#00e5ff'
CYAN_FILL      = '#005f80'
STRAT_BT       = '#4db8ff'
STRAT_LIVE     = '#80ffea'

DD_RED         = '#ff4d6d'
DD_RED_FILL    = '#7b0a2a'
DD_VIOLET      = '#c77dff'
DD_VIOLET_FILL = '#4a0080'
HIGHLIGHT_DD_COLOR = '#ff6347' # Tomato, to highlight worst drawdown strategy

SPARTIACQUE    = pd.Timestamp('2026-02-08')
first_date_str = '2025-01-01' # Modify this date string as needed
first_date     = pd.to_datetime(first_date_str)
cols           = ['date', 'daily_profit', 'current_contracts', 'gap', 'true_range', 'trades']
pattern        = '/content/nikade/*.txt'

DONUT_PALETTE = [
    '#00e5ff', '#2196f3', '#00b4d8', '#7b2fff',
    '#4dd0e1', '#1565c0', '#26c6da', '#5e35b1',
    '#40c4ff', '#0288d1', '#80d8ff', '#304ffe',
    '#18ffff', '#0097a7', '#b388ff', '#1a237e',
]
LOSS_PALETTE = [
    '#ff4d6d', '#c77dff', '#ff6b35', '#e63946',
    '#bd00ff', '#ff0054', '#9b2226', '#6a0572',
    '#ff758c', '#c9184a', '#b5179e', '#7b2d8b',
    '#ff9a00', '#d62828', '#8338ec', '#560bad',
]

# Get today's date for title
today_date = datetime.now().strftime('%d/%m/%Y')

# ━━━━━━━━━━━━━━━━━━━
#  CARICAMENTO DATI
# ━━━━━━━━━━━━━━━━━━━
cumulative_df = pd.DataFrame(columns=cols + ['cum_profit'])
dfs_equity    = []
dfs_risk      = []

print(f"Searching for files with pattern: {pattern}")
found_files = glob.glob(pattern)
print(f"Files found by glob: {found_files}")

for file in found_files:
    df    = pd.read_csv(file, header=None, names=cols, sep=r'\s+', on_bad_lines='skip',
                        parse_dates=['date'], dayfirst=True)
    df    = df[df['date'] >= first_date]
    label = os.path.splitext(os.path.basename(file))[0]

    df_eq               = df.copy()
    df_eq['cum_profit'] = df_eq['daily_profit'].cumsum()
    dfs_equity.append((label, df_eq))
    cumulative_df       = pd.concat([cumulative_df, df_eq])

    df_r               = df.copy()
    df_r['label']      = label
    df_r['cum_profit'] = df_r['daily_profit'].cumsum()
    df_r['cum_max']    = df_r['cum_profit'].cummax()
    df_r['drawdown']   = df_r['cum_profit'] - df_r['cum_max']
    dfs_risk.append(df_r)

# Check if any data was loaded
if not dfs_equity or not dfs_risk:
    print(f"No data files found matching pattern: {pattern}")
    print("Please ensure that the directory 'content/nikade' exists and contains .txt files.")
    # Exit or handle the empty case further down if needed
    # For now, we'll let subsequent code potentially raise errors to show missing data
    # raise ValueError("No objects to concatenate due to missing files.") # Removed to avoid stopping execution

# ══ Portafoglio aggregato ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Ensure dfs_risk is not empty before concatenation
if not dfs_risk:
    print("No dataframes in dfs_risk to concatenate for ptf_risk. Skipping ptf_risk calculation.")
    ptf_risk = pd.DataFrame(columns=['date', 'cum_profit', 'cum_max', 'drawdown'])
else:
    ptf_risk             = pd.concat(dfs_risk).groupby('date')['daily_profit'].sum().cumsum().reset_index()
    ptf_risk.columns     = ['date', 'cum_profit']
    ptf_risk['cum_max']  = ptf_risk['cum_profit'].cummax()
    ptf_risk['drawdown'] = ptf_risk['cum_profit'] - ptf_risk['cum_max']

# Ensure cumulative_df is not empty before calculation
if cumulative_df.empty:
    print("cumulative_df is empty. Skipping result_df calculation.")
    result_df = pd.DataFrame(columns=['date', 'cum_profit', 'cum_max', 'PERDITE'])
else:
    result_df            = cumulative_df.groupby('date')['cum_profit'].sum().reset_index()
    result_df['cum_max'] = result_df['cum_profit'].cummax()
    result_df['PERDITE'] = result_df['cum_profit'] - result_df['cum_max']


# Calculate worst individual strategy drawdown
min_drawdown_overall = float('inf')
worst_strategy_label = None
worst_strategy_df_global = None

if not dfs_risk:
    print("No strategies found, cannot calculate worst individual strategy drawdown.")
else:
    for df_strat in dfs_risk:
        current_min_dd = df_strat['drawdown'].min()
        if current_min_dd < min_drawdown_overall:
            min_drawdown_overall = current_min_dd
            worst_strategy_label = df_strat['label'].iloc[0]
            worst_strategy_df_global = df_strat

# ══ Contributo per strategia ━━━━━━━━━━━━━━━━━━━━━━━
strategy_totals = {}
strategy_losses = {}
for label, df in dfs_equity:
    strategy_totals[label] = df['cum_profit'].iloc[-1] if not df.empty else 0
for df in dfs_risk:
    if not df.empty:
        lbl = df['label'].iloc[0]
        strategy_losses[lbl] = abs(df['drawdown'].min())

profit_labels = [k for k, v in strategy_totals.items() if v > 0]
profit_values = [strategy_totals[k] for k in profit_labels]
loss_labels   = list(strategy_losses.keys())
loss_values   = [strategy_losses[k] for k in loss_labels]

# ══ Split pre/post spartiacque ━━━━━━━━━━━━━━━━━
def split_at(df, date_col='date'):
    if df.empty:
        return pd.DataFrame(), pd.DataFrame()
    mask = df[date_col] <= SPARTIACQUE
    pre  = df[mask]
    post = df[~mask]
    if not pre.empty and not post.empty:
        post = pd.concat([pre.iloc[[-1]], post])
    return pre, post

res_pre, res_post = split_at(result_df)
ptf_pre, ptf_post = split_at(ptf_risk)

# ━━━━━━━━━━━━━━━━━
#  LAYOUT
# ━━━━━━━━━━━━━━━━━
fig = plt.figure(figsize=(18, 26), facecolor=BG_FIGURE)

gs_top = gridspec.GridSpec(4, 1, figure=fig,
                            left=0.07, right=0.97,
                            top=0.97, bottom=0.38,
                            hspace=0.25, # Reverted hspace to previous working value
                            height_ratios=[3, 1, 3, 1])

# NEW: Gridspec for the main title for the donut charts section
gs_donut_section_title = gridspec.GridSpec(1, 1, figure=fig,
                                            left=0.07, right=0.97,
                                            top=0.35, bottom=0.31) # Adjusted coordinates

ax_donut_section_title = fig.add_subplot(gs_donut_section_title[0])
ax_donut_section_title.set_facecolor(BG_FIGURE) # Match figure background
ax_donut_section_title.set_xticks([])
ax_donut_section_title.set_yticks([])
for spine in ax_donut_section_title.spines.values():
    spine.set_visible(False)
ax_donut_section_title.set_title('NIKADE QUANT ─ Analisi contributo per strategia',
                                  fontsize=15, fontweight='bold', pad=0,
                                  color=TITLE_COLOR, loc='center')

gs_bottom = gridspec.GridSpec(1, 2, figure=fig,
                               left=0.05, right=0.97,
                               top=0.28, bottom=0.02, # Adjusted top and bottom for smaller donuts and space for title
                               wspace=0.12)

axes = [fig.add_subplot(gs_top[i]) for i in range(4)]
ax_profit_donut = fig.add_subplot(gs_bottom[0])
ax_loss_donut   = fig.add_subplot(gs_bottom[1])

# ══ Styling assi ━━━━━━━━━━━━━━━━━━━━━━
def style_ax(ax):
    ax.set_facecolor(BG_AXES)
    for spine in ax.spines.values():
        spine.set_color(SPINE_COLOR)
        spine.set_linewidth(0.9)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)
    ax.tick_params(axis='both', colors=TICK_COLOR, labelsize=9, length=4)
    ax.yaxis.label.set_color(LABEL_COLOR)
    ax.grid(axis='y', color=GRID_COLOR, linewidth=0.7, linestyle='--', alpha=0.9)
    ax.grid(axis='x', color=GRID_COLOR, linewidth=0.4, linestyle=':', alpha=0.5)
    for lbl in ax.get_xticklabels() + ax.get_yticklabels():
        lbl.set_color(TICK_COLOR)

for ax in axes:
    style_ax(ax)

def add_spartiacque_line(ax, label=False):
    ax.axvline(SPARTIACQUE, color='#4a90d9', linewidth=1.3,
               linestyle='--', alpha=0.9, zorder=10)
    if label:
        ylim = ax.get_ylim()
        ypos = ylim[0] + (ylim[1] - ylim[0]) * 0.93
        ax.text(SPARTIACQUE, ypos,
                '  08 Feb 2026\n  Live ─ no edit',
                fontsize=8, color='#a0ccff', va='top', fontstyle='italic')

# ━━━━━━━━━━━━━━━━━
#  [0] EQUITY LINE
# ━━━━━━━━━━━━━━━━━
ax0 = axes[0]
if result_df.empty:
    ax0.text(0.5, 0.5, 'Nessun dato (Equity Line)', ha='center', va='center',
             color=TEXT_WHITE, fontsize=12, transform=ax0.transAxes)
else:
    for lbl, df in dfs_equity:
        pre      = df[df['date'] <= SPARTIACQUE]
        post_idx = df[df['date'] >  SPARTIACQUE]
        if not pre.empty and not post_idx.empty:
            post_idx = pd.concat([pre.iloc[[-1]], post_idx])
        ax0.plot(pre['date'],      pre['cum_profit'],      lw=0.9, alpha=0.45, color=STRAT_BT)
        ax0.plot(post_idx['date'], post_idx['cum_profit'], lw=0.9, alpha=0.45, color=STRAT_LIVE)

    ax0.fill_between(res_pre['date'],  0, res_pre['cum_profit'],  color=BLUE_FILL, alpha=0.35)
    ax0.fill_between(res_post['date'], 0, res_post['cum_profit'], color=CYAN_FILL, alpha=0.40)
    ax0.plot(res_pre['date'],  res_pre['cum_profit'],  color=BLUE_LINE, lw=2.8)
    ax0.plot(res_post['date'], res_post['cum_profit'], color=CYAN_LINE, lw=3.0)

    junc_val = res_pre['cum_profit'].iloc[-1] if not res_pre.empty else 0
    ax0.scatter([SPARTIACQUE], [junc_val], color=BG_AXES,
                edgecolors=CYAN_LINE, zorder=7, s=90, linewidths=2.5)

    max_profit = result_df['cum_profit'].max()
    max_date   = result_df.loc[result_df['cum_profit'].idxmax(), 'date']
    ax0.annotate(
        f'Max Profit: ${max_profit:,.0f}\n{max_date.strftime("%d %b %Y")}',
        xy=(max_date, max_profit),
        xytext=(max_date, max_profit * 0.72),
        arrowprops=dict(arrowstyle='->', color=CYAN_LINE, lw=1.6),
        color=TEXT_WHITE, fontsize=9.5, fontweight='bold',
        bbox=dict(boxstyle='round,pad=0.4', facecolor='#062050',
                  edgecolor=CYAN_LINE, alpha=0.92, linewidth=1.2)
    )
    ax0.scatter([max_date], [max_profit], color=CYAN_LINE, zorder=6, s=75)

ax0.set_ylabel('Curva dei Profitti ($)', fontsize=10.5, labelpad=10)
ax0.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'${x:,.0f}'))
ax0.set_ylim(bottom=0)
ax0.set_title(f'NIKADE QUANT  ─  Equity Line ({today_date})',
              fontsize=15, fontweight='bold', pad=20,
              color=TITLE_COLOR, loc='center') # Centered and increased pad

for lbl in ax0.get_yticklabels(): lbl.set_color(TICK_COLOR)

patch_bt   = mpatches.Patch(color=BLUE_LINE,  label='Portafoglio backtest')
patch_live = mpatches.Patch(color=CYAN_LINE,  label='Portafoglio live (no edit)')
patch_sl   = mpatches.Patch(color=STRAT_BT,   alpha=0.6, label='Strategie singole (bt)')
patch_sl2  = mpatches.Patch(color=STRAT_LIVE, alpha=0.6, label='Strategie singole (live)')
ax0.legend(handles=[patch_bt, patch_live, patch_sl, patch_sl2],
           fontsize=9, loc='upper left', framealpha=0.7,
           facecolor='#051535', edgecolor='#163580', labelcolor=TEXT_WHITE)

add_spartiacque_line(ax0, label=True)

# ━━━━━━━━━━━━━━━━━
#  [1] DRAWDOWN PORTAFOGLIO
# ━━━━━━━━━━━━━━━━━
ax1 = axes[1]
if result_df.empty:
    ax1.text(0.5, 0.5, 'Nessun dato (Drawdown Portafoglio)', ha='center', va='center',
             color=TEXT_WHITE, fontsize=12, transform=ax1.transAxes)
else:
    dd_pre      = result_df[result_df['date'] <= SPARTIACQUE]
    dd_post_raw = result_df[result_df['date'] >  SPARTIACQUE]
    dd_post = pd.concat([dd_pre.iloc[[-1]], dd_post_raw]) if (not dd_pre.empty and not dd_post_raw.empty) else dd_post_raw

    ax1.fill_between(dd_pre['date'],  0, dd_pre['PERDITE'],  color=DD_RED_FILL,    alpha=0.80)
    ax1.fill_between(dd_post['date'], 0, dd_post['PERDITE'], color=DD_VIOLET_FILL, alpha=0.80)
    ax1.plot(dd_pre['date'],  dd_pre['PERDITE'],  color=DD_RED,    lw=1.4)
    ax1.plot(dd_post['date'], dd_post['PERDITE'], color=DD_VIOLET, lw=1.6)

ax1.set_ylabel('Drawdown ($)', fontsize=10, labelpad=10)
ax1.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'${x:,.0f}'))
ax1.set_ylim(top=0)
ax1.axhline(0, color='#4a90d9', linewidth=0.8, linestyle='--')
for lbl in ax1.get_yticklabels(): lbl.set_color(TICK_COLOR)

pd1 = mpatches.Patch(color=DD_RED,    label='Drawdown backtest')
pd2 = mpatches.Patch(color=DD_VIOLET, label='Drawdown live')
ax1.legend(handles=[pd1, pd2], fontsize=8.5, loc='lower left',
           framealpha=0.7, facecolor='#051535', edgecolor='#163580', labelcolor=TEXT_WHITE)
add_spartiacque_line(ax1)

# ━━━━━━━━━━━━━━━━━
#  [2] DRAWDOWN STRATEGIE
# ━━━━━━━━━━━━━━━━━
ax2 = axes[2]
if not dfs_risk:
    ax2.text(0.5, 0.5, 'Nessun dato (Drawdown Strategie)', ha='center', va='center',
             color=TEXT_WHITE, fontsize=12, transform=ax2.transAxes)
else:
    for df_strat in dfs_risk: # Renamed loop variable to avoid conflict
        if not df_strat.empty:
            is_worst_strategy = (df_strat['label'].iloc[0] == worst_strategy_label)

            line_color_bt   = HIGHLIGHT_DD_COLOR if is_worst_strategy else STRAT_BT
            line_color_live = HIGHLIGHT_DD_COLOR if is_worst_strategy else STRAT_LIVE
            line_width      = 2.0 if is_worst_strategy else 0.9
            alpha_value     = 0.9 if is_worst_strategy else 0.50

            pre      = df_strat[df_strat['date'] <= SPARTIACQUE]
            post_idx = df_strat[df_strat['date'] >  SPARTIACQUE]
            if not pre.empty and not post_idx.empty:
                post_idx = pd.concat([pre.iloc[[-1]], post_idx])

            ax2.plot(pre['date'],      pre['drawdown'],      lw=line_width, alpha=alpha_value, color=line_color_bt)
            ax2.plot(post_idx['date'], post_idx['drawdown'], lw=line_width, alpha=alpha_value, color=line_color_live)

    if worst_strategy_df_global is not None and min_drawdown_overall != float('inf'):
        date_worst_dd = worst_strategy_df_global.loc[worst_strategy_df_global['drawdown'].idxmin(), 'date']
        ax2.scatter([date_worst_dd], [min_drawdown_overall], color=HIGHLIGHT_DD_COLOR, zorder=7, s=100, marker='X')
        ax2.annotate(
            f'Max DD: ${min_drawdown_overall:,.0f}\nStrategy: {worst_strategy_label}',
            xy=(date_worst_dd, min_drawdown_overall),
            xytext=(date_worst_dd, min_drawdown_overall - (abs(min_drawdown_overall) * 0.2)), # Adjusted vertical position
            arrowprops=dict(arrowstyle='->', color=HIGHLIGHT_DD_COLOR, lw=1.5),
            color=TEXT_WHITE, fontsize=9.5, fontweight='bold',
            bbox=dict(boxstyle='round,pad=0.35', facecolor='#3a0020',
                  edgecolor=HIGHLIGHT_DD_COLOR, alpha=0.90, linewidth=1.1)
        )

ax2.axhline(0, color='#4a90d9', lw=0.8, linestyle='--')
ax2.set_ylabel('Drawdown Strategie ($)', fontsize=10.5, labelpad=10)
ax2.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'${x:,.0f}'))
ax2.set_title('NIKADE QUANT  ─  Andamento del Rischio',
              fontsize=15, fontweight='bold', pad=20,
              color=TITLE_COLOR, loc='center') # Centered and increased pad
for lbl in ax2.get_yticklabels(): lbl.set_color(TICK_COLOR)

add_spartiacque_line(ax2, label=True)

# ━━━━━━━━━━━━━━━━━
#  [3] DRAWDOWN PORTAFOGLIO TOTALE
# ━━━━━━━━━━━━━━━━━
ax3 = axes[3]
if ptf_risk.empty:
    ax3.text(0.5, 0.5, 'Nessun dato (Drawdown Portafoglio Totale)', ha='center', va='center',
             color=TEXT_WHITE, fontsize=12, transform=ax3.transAxes)
else:
    pr_pre      = ptf_risk[ptf_risk['date'] <= SPARTIACQUE]
    pr_post_raw = ptf_risk[ptf_risk['date'] >  SPARTIACQUE]
    pr_post = pd.concat([pr_pre.iloc[[-1]], pr_post_raw]) if (not pr_pre.empty and not pr_post_raw.empty) else pr_post_raw

    ax3.fill_between(pr_pre['date'],  0, pr_pre['drawdown'],  color=DD_RED_FILL,    alpha=0.80)
    ax3.fill_between(pr_post['date'], 0, pr_post['drawdown'], color=DD_VIOLET_FILL, alpha=0.80)
    ax3.plot(pr_pre['date'],  pr_pre['drawdown'],  color=DD_RED,    lw=1.4)
    ax3.plot(pr_post['date'], pr_post['drawdown'], color=DD_VIOLET, lw=1.6)

    min_dd   = ptf_risk['drawdown'].min()
    min_date = ptf_risk.loc[ptf_risk['drawdown'].idxmin(), 'date']
    ax3.annotate(
        f'Max DD: ${min_dd:,.0f}',
        xy=(min_date, min_dd),
        xytext=(min_date, min_dd * 0.55),
        arrowprops=dict(arrowstyle='->', color=DD_RED, lw=1.5),
        color=TEXT_WHITE, fontsize=9.5, fontweight='bold',
        bbox=dict(boxstyle='round,pad=0.35', facecolor='#3a0020',
                  edgecolor=DD_RED, alpha=0.90, linewidth=1.1)
    )

ax3.axhline(0, color='#4a90d9', lw=0.8, linestyle='--')
ax3.set_ylabel('Drawdown Portafoglio ($)', fontsize=10, labelpad=10)
ax3.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'${x:,.0f}'))
for lbl in ax3.get_yticklabels(): lbl.set_color(TICK_COLOR)

add_spartiacque_line(ax3)

# ━━━━━━━━━━━━━━━━━
#  HELPER CIAMBELLA
# ━━━━━━━━━━━━━━━━━
def draw_donut(ax, values, labels, colors, title,
               center_label, center_sub, center_color):
    ax.set_facecolor(BG_DONUT)
    total = sum(values)
    if total == 0:
        ax.text(0.5, 0.5, 'Nessun dato', ha='center', va='center',
                color=TEXT_WHITE, fontsize=12, transform=ax.transAxes)
        # Hide legend if no data
        if ax.get_legend() is not None:
            ax.get_legend().remove()
        return

    wedges, _ = ax.pie(
        values,
        labels=None,
        colors=colors[:len(values)],
        startangle=90,
        counterclock=False,
        wedgeprops=dict(width=0.52, edgecolor=BG_FIGURE, linewidth=1.8),
    )

    # Percentuali nei settori > 5%
    for wedge, val in zip(wedges, values):
        pct = val / total * 100
        if pct >= 5:
            angle     = (wedge.theta2 + wedge.theta1) / 2
            angle_rad = np.deg2rad(angle)
            r = 0.74
            x = r * np.cos(angle_rad)
            y = r * np.sin(angle_rad)
            ax.text(x, y, f'{pct:.1f}%', ha='center', va='center',
                    fontsize=8.5, fontweight='bold', color=TEXT_WHITE, zorder=5)

    # Centro
    ax.text(0,  0.12, center_label, ha='center', va='center',
            fontsize=13, fontweight='bold', color=center_color)
    ax.text(0, -0.16, center_sub,  ha='center', va='center',
            fontsize=8.5, color='#a0ccff')

    ax.set_title(title, fontsize=12, fontweight='bold',
                 color=TITLE_COLOR, pad=18)

    # Legenda
    legend_elements = []
    for lbl, val, col in zip(labels, values, colors):
        pct   = val / total * 100
        short = lbl[:18] + '…' if len(lbl) > 18 else lbl
        legend_elements.append(
            mpatches.Patch(color=col,
                           label=f'{short:<20}  ${val:>10,.0f}   {pct:5.1f}%')
        )

    ax.legend(
        handles=legend_elements,
        loc='lower center',
        bbox_to_anchor=(0.5, -0.52),
        fontsize=10, ncol=2, # Increased legend fontsize
        framealpha=0.85,
        facecolor='#051535',
        edgecolor='#163580',
        labelcolor=TEXT_WHITE,
        handlelength=1.2,
        handleheight=1.0,
        columnspacing=1.2,
        handletextpad=0.6,
    )

# ━━━━━━━━━━━━━━━━━
#  [4a] CIAMBELLA PROFITTI
# ━━━━━━━━━━━━━━━━━
total_profit = sum(profit_values) if profit_values else 0
draw_donut(
    ax=ax_profit_donut,
    values=profit_values,
    labels=profit_labels,
    colors=DONUT_PALETTE,
    title='Contributo al Profitto per Strategia',
    center_label=f'${total_profit:,.0f}',
    center_sub='Profitto totale',
    center_color=CYAN_LINE,
)

# ━━━━━━━━━━━━━━━━━
#  [4b] CIAMBELLA PERDITE
# ━━━━━━━━━━━━━━━━━
total_loss = sum(loss_values) if loss_values else 0
draw_donut(
    ax=ax_loss_donut,
    values=loss_values,
    labels=loss_labels,
    colors=LOSS_PALETTE,
    title='Contributo alle Perdite ─ Max Drawdown per Strategia',
    center_label=f'${total_loss:,.0f}',
    center_sub='Perdita cumulata totale',
    center_color=DD_RED,
)

# ══ Separatore ━━━━━━━━━━━━━━━━
fig.add_artist(plt.Line2D(
    [0.04, 0.96], [0.295, 0.295], # Adjusted y position to be between new title and donuts
    transform=fig.transFigure,
    color='#163580', linewidth=1.0, linestyle='--', alpha=0.9
))

# ══ Fix tick colori x su tutti gli assi ━━━━━━━━━━━━━━━
for ax in axes:
    for lbl in ax.get_xticklabels():
        lbl.set_color(TICK_COLOR)
        lbl.set_fontsize(9)

fig.tight_layout() # Re-enabled for better overall layout
plt.savefig('nikade_chart.png', dpi=150, facecolor=BG_FIGURE, bbox_inches='tight')
print("Salvato: nikade_chart.png")
plt.show()
