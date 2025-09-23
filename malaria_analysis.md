## Code
```python
"""
Malaria Testing Epidemiological Trend Analysis - PNG, PDF, and HTML Output with Individual Y-Axis Scaling
Creates individual PNG files, multi-page PDFs, and HTML reports with Enhanced Collapsible Navigation
Variables: Microscopy and RDT testing (positive/negative) across age groups and health facility setting
Order: National → District Subplots → Individual Districts → Chiefdom Subplots
EACH PLOT HAS ITS OWN INDIVIDUAL Y-AXIS SCALING BASED ON ITS DATA VALUES
Shows actual numbers (not percentages) with proper scaling
ENHANCED VERSION: ALL PLOTS GUARANTEED TO HAVE LEGENDS
"""

import os
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np
import re
from pathlib import Path
import zipfile
from matplotlib.backends.backend_pdf import PdfPages
import base64
from io import BytesIO
from datetime import datetime

def create_malaria_testing_analysis(input_file="/content/comparative_analysis.xlsx",
                                   output_base_dir='malaria_testing_plots/',
                                   main_title="Malaria Testing Results",
                                   y_axis_label="Number of tests"):
    """Create comprehensive malaria testing analysis with PNG files, PDFs, and HTML with individual y-axis scaling"""

    print("🚀 Starting Malaria Testing Analysis - Microscopy and RDT Results...")
    print("=" * 70)

    # Variables to analyze - Malaria testing results by method, result, and age group
    variables = [
        'test_neg_mic_u5_hf',      # Microscopy Negative - Under 5 - HF
        'test_pos_mic_u5_hf',      # Microscopy Positive - Under 5 - HF
        'test_neg_mic_5_14_hf',    # Microscopy Negative - 5-14 - HF
        'test_pos_mic_5_14_hf',    # Microscopy Positive - 5-14 - HF
        'test_neg_mic_ov15_hf',    # Microscopy Negative - Over 15 - HF
        'test_pos_mic_ov15_hf',    # Microscopy Positive - Over 15 - HF
        'test_neg_rdt_u5_hf',      # RDT Negative - Under 5 - HF
        'test_pos_rdt_u5_hf',      # RDT Positive - Under 5 - HF
        'test_neg_rdt_5_14_hf',    # RDT Negative - 5-14 - HF
        'test_pos_rdt_5_14_hf',    # RDT Positive - 5-14 - HF
        'test_neg_rdt_ov15_hf',    # RDT Negative - Over 15 - HF
        'test_pos_rdt_ov15_hf'     # RDT Positive - Over 15 - HF
    ]

    # Colors for different test types and results - DISTINCT COLORS
    colors = [
        '#1f77b4',  # Blue - Mic Neg U5
        '#ff7f0e',  # Orange - Mic Pos U5
        '#2ca02c',  # Green - Mic Neg 5-14
        '#d62728',  # Red - Mic Pos 5-14
        '#9467bd',  # Purple - Mic Neg Over 15
        '#8c564b',  # Brown - Mic Pos Over 15
        '#e377c2',  # Pink - RDT Neg U5
        '#7f7f7f',  # Gray - RDT Pos U5
        '#bcbd22',  # Olive - RDT Neg 5-14
        '#17becf',  # Cyan - RDT Pos 5-14
        '#ff9896',  # Light Red - RDT Neg Over 15
        '#aec7e8'   # Light Blue - RDT Pos Over 15
    ]

    # Labels for legend
    labels = [
        'Mic Neg U5',
        'Mic Pos U5',
        'Mic Neg 5-14',
        'Mic Pos 5-14',
        'Mic Neg O15',
        'Mic Pos O15',
        'RDT Neg U5',
        'RDT Pos U5',
        'RDT Neg 5-14',
        'RDT Pos 5-14',
        'RDT Neg O15',
        'RDT Pos O15'
    ]

    # Target years (2021-2024)
    target_years = [2021, 2022, 2023, 2024]

    # Load data once
    print("📊 Loading data...")
    try:
        df = pd.read_excel(input_file)
        print(f"✓ Data loaded: {len(df)} rows")
        print(f"✓ Available columns: {list(df.columns)}")
    except FileNotFoundError:
        print(f"❌ Error: {input_file} not found")
        return

    # Check required columns
    required_cols = ['FIRST_DNAM', 'FIRST_CHIE']
    missing_cols = [col for col in required_cols if col not in df.columns]
    if missing_cols:
        print(f"❌ Error: Missing required columns: {missing_cols}")
        return

    # Create main output directory and subdirectories
    os.makedirs(output_base_dir, exist_ok=True)
    os.makedirs(f'{output_base_dir}/png_files/', exist_ok=True)
    os.makedirs(f'{output_base_dir}/pdf_files/', exist_ok=True)

    # Store all figures for HTML generation
    html_images = []
    all_png_files = []

    def save_and_track_figure(fig, filename, title, variable, section_type="", save_png=True):
        """Save figure as PNG and track for HTML with section information"""
        if save_png:
            png_path = f'{output_base_dir}/png_files/{filename}.png'
            fig.savefig(png_path, dpi=300, bbox_inches='tight')
            all_png_files.append(png_path)
            print(f"   ✓ Saved PNG: {filename}.png")

        # Convert to base64 for HTML
        buffer = BytesIO()
        fig.savefig(buffer, format='png', dpi=150, bbox_inches='tight')
        buffer.seek(0)
        image_base64 = base64.b64encode(buffer.getvalue()).decode('utf-8')
        buffer.close()

        html_images.append({
            'title': title,
            'image': image_base64,
            'variable': variable,
            'filename': filename,
            'section_type': section_type
        })
        return image_base64

    def get_year_columns_for_prefix(prefix):
        """Get year columns for a specific prefix (e.g., 'test_neg_mic_u5_hf')"""
        pattern = re.compile(f'^{prefix}_(\d{{4}})$')
        year_cols = []
        for col in df.columns:
            match = pattern.match(col)
            if match:
                year = int(match.group(1))
                if year in target_years:
                    year_cols.append(col)
        return sorted(year_cols)

    def calculate_individual_y_limits(data_values):
        """Calculate y-axis limits for individual plot data"""
        if len(data_values) == 0:
            return 0, 100  # Default fallback

        # Filter out NaN values
        clean_values = [v for v in data_values if not pd.isna(v)]

        if len(clean_values) == 0:
            return 0, 100

        y_min = 0  # Always start from 0
        y_max = max(clean_values) * 1.1  # Add 10% padding

        # Ensure minimum range
        if y_max < 10:
            y_max = 10

        return y_min, y_max

    def add_comprehensive_legend(ax, legend_handles, legend_labels, position='upper left', bbox=(1.02, 1), fontsize=8):
        """Add comprehensive legend to any plot, ensuring it's always visible"""
        if legend_handles and legend_labels:
            legend = ax.legend(legend_handles, legend_labels,
                             fontsize=fontsize,
                             loc=position,
                             bbox_to_anchor=bbox,
                             frameon=True,
                             fancybox=True,
                             shadow=True,
                             framealpha=0.9,
                             facecolor='white',
                             edgecolor='gray',
                             borderpad=0.5)
            # Make legend text more readable
            for text in legend.get_texts():
                text.set_fontweight('bold')
            return legend
        return None

    # Get districts list
    districts = df['FIRST_DNAM'].dropna().unique()

    # ==========================================
    # CREATE COMBINED ANALYSIS (ALL VARIABLES ON ONE PDF)
    # ==========================================
    print(f"\n{'='*70}")
    print(f"📈 CREATING MALARIA TESTING ANALYSIS PDF")
    print(f"{'='*70}")

    pdf_filename = f'{output_base_dir}/pdf_files/malaria_testing_analysis.pdf'
    print(f"📄 Creating PDF: {pdf_filename}")

    with PdfPages(pdf_filename) as pdf:

        # ==========================================
        # PAGE 1: NATIONAL TRENDS (ALL TESTING VARIABLES)
        # ==========================================
        print(f"📊 Creating Page 1: National trends - All malaria testing results...")

        fig, ax = plt.subplots(figsize=(16, 10))

        # Calculate individual scaling for national trends
        national_values = []
        legend_handles = []
        legend_labels = []

        for variable, color, label in zip(variables, colors, labels):
            year_cols = get_year_columns_for_prefix(variable)
            if len(year_cols) < 2:
                continue

            years = [int(re.search(r'(\d{4})', col).group(1)) for col in year_cols]

            # Calculate national totals (sum, not average)
            national_total = df[year_cols].sum(axis=0)
            values = national_total.values
            national_values.extend(values[~pd.isna(values)])

            if not pd.isna(values).all():
                line = ax.plot(years, values, marker='o', linewidth=2, markersize=6,
                              color=color, label=label)[0]
                legend_handles.append(line)
                legend_labels.append(label)

        # Calculate and apply individual y-limits for national trends
        if national_values:
            national_y_min, national_y_max = calculate_individual_y_limits(national_values)
        else:
            national_y_min, national_y_max = 0, 100

        ax.set_title(f'National {main_title} Trends (2021-2024)',
                    fontsize=16, fontweight='bold', pad=20)
        ax.set_xlabel('Year', fontweight='bold', fontsize=14)
        ax.set_ylabel(y_axis_label, fontweight='bold', fontsize=14)
        ax.grid(True, alpha=0.3)
        ax.set_xticks(target_years)
        ax.tick_params(labelsize=10)
        ax.set_ylim(national_y_min, national_y_max)

        # Format y-axis to show actual numbers with commas
        ax.ticklabel_format(style='plain', axis='y')
        ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'{x:,.0f}'))

        # ENHANCED: Always add legend
        add_comprehensive_legend(ax, legend_handles, legend_labels, 'upper left', (1.02, 1), 8)

        plt.tight_layout()
        pdf.savefig(fig, bbox_inches='tight')
        save_and_track_figure(fig, 'malaria_01_national_all',
                            f'National {main_title} - All Testing Results', 'combined', 'national')
        plt.close(fig)

        # ==========================================
        # PAGE 2: MICROSCOPY vs RDT COMPARISON
        # ==========================================
        print(f"🔬 Creating Page 2: Microscopy vs RDT comparison...")

        fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(20, 8))
        fig.suptitle('Microscopy vs RDT Testing Results Comparison',
                     fontsize=16, fontweight='bold', y=0.95)

        # Microscopy variables
        mic_variables = [var for var in variables if 'mic' in var]
        mic_colors = [colors[i] for i, var in enumerate(variables) if 'mic' in var]
        mic_labels = [labels[i] for i, var in enumerate(variables) if 'mic' in var]
        mic_values = []
        mic_legend_handles = []
        mic_legend_labels = []

        for variable, color, label in zip(mic_variables, mic_colors, mic_labels):
            year_cols = get_year_columns_for_prefix(variable)
            if not year_cols:
                continue

            years = [int(re.search(r'(\d{4})', col).group(1)) for col in year_cols]
            national_total = df[year_cols].sum(axis=0)
            values = national_total.values
            mic_values.extend(values[~pd.isna(values)])

            if not pd.isna(values).all():
                line = ax1.plot(years, values, marker='o', linewidth=3, markersize=5,
                               color=color, label=label)[0]
                mic_legend_handles.append(line)
                mic_legend_labels.append(label)

        if mic_values:
            mic_y_min, mic_y_max = calculate_individual_y_limits(mic_values)
            ax1.set_ylim(mic_y_min, mic_y_max)

        ax1.set_title('Microscopy Testing', fontsize=14, fontweight='bold', pad=20)
        ax1.set_xlabel('Year', fontweight='bold')
        ax1.set_ylabel(y_axis_label, fontweight='bold')
        ax1.grid(True, alpha=0.3)
        ax1.set_xticks(target_years)
        ax1.ticklabel_format(style='plain', axis='y')
        ax1.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'{x:,.0f}'))

        # ENHANCED: Always add legend for microscopy
        add_comprehensive_legend(ax1, mic_legend_handles, mic_legend_labels, 'upper left', (1.02, 1), 7)

        # RDT variables
        rdt_variables = [var for var in variables if 'rdt' in var]
        rdt_colors = [colors[i] for i, var in enumerate(variables) if 'rdt' in var]
        rdt_labels = [labels[i] for i, var in enumerate(variables) if 'rdt' in var]
        rdt_values = []
        rdt_legend_handles = []
        rdt_legend_labels = []

        for variable, color, label in zip(rdt_variables, rdt_colors, rdt_labels):
            year_cols = get_year_columns_for_prefix(variable)
            if not year_cols:
                continue

            years = [int(re.search(r'(\d{4})', col).group(1)) for col in year_cols]
            national_total = df[year_cols].sum(axis=0)
            values = national_total.values
            rdt_values.extend(values[~pd.isna(values)])

            if not pd.isna(values).all():
                line = ax2.plot(years, values, marker='o', linewidth=3, markersize=5,
                               color=color, label=label)[0]
                rdt_legend_handles.append(line)
                rdt_legend_labels.append(label)

        if rdt_values:
            rdt_y_min, rdt_y_max = calculate_individual_y_limits(rdt_values)
            ax2.set_ylim(rdt_y_min, rdt_y_max)

        ax2.set_title('RDT Testing', fontsize=14, fontweight='bold', pad=20)
        ax2.set_xlabel('Year', fontweight='bold')
        ax2.set_ylabel(y_axis_label, fontweight='bold')
        ax2.grid(True, alpha=0.3)
        ax2.set_xticks(target_years)
        ax2.ticklabel_format(style='plain', axis='y')
        ax2.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'{x:,.0f}'))

        # ENHANCED: Always add legend for RDT
        add_comprehensive_legend(ax2, rdt_legend_handles, rdt_legend_labels, 'upper left', (1.02, 1), 7)

        plt.tight_layout()
        pdf.savefig(fig, bbox_inches='tight')
        save_and_track_figure(fig, 'malaria_02_mic_vs_rdt',
                            f'{main_title} - Microscopy vs RDT', 'combined', 'comparison')
        plt.close(fig)

        # ==========================================
        # PAGE 3: DISTRICT SUBPLOTS (ALL VARIABLES)
        # ==========================================
        print(f"🏘️ Creating Page 3: District subplots with individual scaling...")

        # Calculate grid dimensions
        n_districts = len(districts)
        n_cols = 3
        n_rows = int(np.ceil(n_districts / n_cols))

        fig, axes = plt.subplots(n_rows, n_cols, figsize=(20, 5 * n_rows))
        fig.suptitle(f'{main_title} - All Districts Overview (2021-2024)',
                     fontsize=16, fontweight='bold', y=0.98)

        # Handle different subplot configurations
        if n_districts == 1:
            axes = [axes]
        elif n_rows == 1:
            axes = [axes] if n_districts == 1 else axes
        else:
            axes = axes.flatten()

        for i, district in enumerate(districts):
            if i >= len(axes):
                break

            ax = axes[i] if isinstance(axes, list) else axes[i]
            district_data = df[df['FIRST_DNAM'] == district]

            # Collect values for THIS district's individual scaling
            district_values = []

            for variable, color, label in zip(variables, colors, labels):
                year_cols = get_year_columns_for_prefix(variable)
                if not year_cols:
                    continue

                years = [int(re.search(r'(\d{4})', col).group(1)) for col in year_cols]
                district_total = district_data[year_cols].sum(axis=0)
                values = district_total.values
                district_values.extend(values[~pd.isna(values)])

                if not pd.isna(values).all():
                    ax.plot(years, values, marker='o', linewidth=1.5, markersize=2,
                           color=color, alpha=0.8)

            # Calculate and apply individual scaling for THIS district
            if district_values:
                district_y_min, district_y_max = calculate_individual_y_limits(district_values)
            else:
                district_y_min, district_y_max = 0, 100

            ax.set_title(district, fontsize=10, fontweight='bold')
            ax.set_xticks(target_years)
            ax.tick_params(labelsize=8)
            ax.grid(True, alpha=0.3)
            ax.set_ylim(district_y_min, district_y_max)
            ax.ticklabel_format(style='plain', axis='y')
            ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'{x:,.0f}'))

        # Hide unused subplots
        if isinstance(axes, list) or hasattr(axes, 'flatten'):
            axes_to_check = axes if isinstance(axes, list) else axes.flatten()
            for i in range(len(districts), len(axes_to_check)):
                axes_to_check[i].set_visible(False)

        # ENHANCED: Add comprehensive legend for district overview
        legend_elements = [plt.Line2D([0], [0], color=color, lw=2, label=label, marker='o', markersize=4)
                         for variable, color, label in zip(variables, colors, labels)]
        fig.legend(handles=legend_elements, loc='center right',
                  bbox_to_anchor=(0.98, 0.5), ncol=1, fontsize=7,
                  frameon=True, fancybox=True, shadow=True,
                  framealpha=0.9, facecolor='white', edgecolor='gray')

        # Add common labels
        fig.text(0.5, 0.02, 'Year', ha='center', fontweight='bold', fontsize=12)
        fig.text(0.02, 0.5, y_axis_label, va='center', rotation='vertical',
                fontweight='bold', fontsize=12)

        plt.tight_layout(rect=[0.03, 0.03, 0.85, 0.96])
        pdf.savefig(fig, bbox_inches='tight')
        save_and_track_figure(fig, 'malaria_03_districts_overview',
                            f'{main_title} - All Districts Overview', 'combined', 'districts_overview')
        plt.close(fig)

        # ==========================================
        # PAGES 4+: INDIVIDUAL DISTRICT PLOTS
        # ==========================================
        print(f"📈 Creating Pages 4+: Individual district plots...")

        for district_idx, district in enumerate(districts, 1):
            print(f"   Creating Page {district_idx + 3}: {district}")

            fig, ax = plt.subplots(figsize=(16, 10))

            district_data = df[df['FIRST_DNAM'] == district]
            legend_handles = []
            legend_labels = []

            # Collect values for this district's individual scaling
            district_plot_values = []

            for variable, color, label in zip(variables, colors, labels):
                year_cols = get_year_columns_for_prefix(variable)
                if not year_cols:
                    continue

                years = [int(re.search(r'(\d{4})', col).group(1)) for col in year_cols]
                district_total = district_data[year_cols].sum(axis=0)
                values = district_total.values
                district_plot_values.extend(values[~pd.isna(values)])

                if not pd.isna(values).all():
                    line = ax.plot(years, values, marker='o', linewidth=2, markersize=5,
                                  color=color, label=label)[0]
                    legend_handles.append(line)
                    legend_labels.append(label)

            # Calculate individual y-limits for this district
            if district_plot_values:
                district_y_min, district_y_max = calculate_individual_y_limits(district_plot_values)
            else:
                district_y_min, district_y_max = 0, 100

            ax.set_title(f'{main_title} Trends - {district}',
                       fontsize=16, fontweight='bold', pad=20)
            ax.set_xlabel('Year', fontweight='bold', fontsize=14)
            ax.set_ylabel(y_axis_label, fontweight='bold', fontsize=14)
            ax.grid(True, alpha=0.3)
            ax.set_xticks(target_years)
            ax.tick_params(labelsize=12)
            ax.set_ylim(district_y_min, district_y_max)
            ax.ticklabel_format(style='plain', axis='y')
            ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'{x:,.0f}'))

            # ENHANCED: Always add legend for individual districts
            add_comprehensive_legend(ax, legend_handles, legend_labels, 'upper left', (1.02, 1), 7)

            plt.tight_layout(rect=[0.05, 0.05, 0.85, 0.95])
            pdf.savefig(fig, bbox_inches='tight')

            # Save as individual PNG
            safe_name = "".join(c for c in district if c.isalnum() or c in (' ', '-', '_')).strip().replace(' ', '_')
            save_and_track_figure(fig, f'malaria_04_{district_idx:02d}_{safe_name}',
                                f'{main_title} - {district}', 'combined', 'individual_district')
            plt.close(fig)

        # ==========================================
        # FINAL PAGES: CHIEFDOM SUBPLOT PAGES
        # ==========================================
        print(f"🏡 Creating final pages: Chiefdom subplots...")

        for district_idx, district in enumerate(districts, 1):
            print(f"   Creating chiefdom subplots for {district}")

            district_data = df[df['FIRST_DNAM'] == district]
            if len(district_data) == 0:
                continue

            chiefdoms = district_data['FIRST_CHIE'].dropna().unique()
            if len(chiefdoms) == 0:
                continue

            # Calculate grid dimensions
            n_cols = 3
            n_rows = int(np.ceil(len(chiefdoms) / n_cols))

            fig, axes = plt.subplots(n_rows, n_cols, figsize=(20, 5 * n_rows))
            page_title = f'{main_title} - {district} (Chiefdoms)'
            fig.suptitle(page_title, fontsize=16, fontweight='bold', y=0.98)

            # Handle different subplot configurations
            if len(chiefdoms) == 1:
                axes = [axes]
            elif n_rows == 1:
                axes = [axes] if len(chiefdoms) == 1 else axes
            else:
                axes = axes.flatten()

            for i, chiefdom in enumerate(chiefdoms):
                if i >= len(axes):
                    break

                ax = axes[i] if isinstance(axes, list) else axes[i]
                chie_data = district_data[district_data['FIRST_CHIE'] == chiefdom]

                if len(chie_data) == 0:
                    continue

                # Collect values for THIS chiefdom's individual scaling
                chiefdom_values = []

                for variable, color, label in zip(variables, colors, labels):
                    year_cols = get_year_columns_for_prefix(variable)
                    if not year_cols:
                        continue

                    years = [int(re.search(r'(\d{4})', col).group(1)) for col in year_cols]

                    # Sum values for each chiefdom
                    chie_total = chie_data[year_cols].sum(axis=0)
                    values = chie_total.values
                    chiefdom_values.extend(values[~pd.isna(values)])

                    if not pd.isna(values).all():
                        ax.plot(years, values, marker='o', color=color, linewidth=1.5, markersize=1,
                               alpha=0.8)

                # Calculate and apply individual scaling for THIS chiefdom
                if chiefdom_values:
                    chie_y_min, chie_y_max = calculate_individual_y_limits(chiefdom_values)
                else:
                    chie_y_min, chie_y_max = 0, 100

                ax.set_title(chiefdom, fontsize=9, fontweight='bold', pad=5)
                ax.set_xticks(target_years)
                ax.tick_params(axis='x', labelsize=8)
                ax.tick_params(axis='y', labelsize=8)
                ax.grid(True, alpha=0.3)
                ax.set_ylim(chie_y_min, chie_y_max)
                ax.ticklabel_format(style='plain', axis='y')
                ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'{x:,.0f}'))

            # Hide unused subplots
            axes_to_check = axes if isinstance(axes, list) or not hasattr(axes, 'flatten') else axes.flatten()
            for j in range(len(chiefdoms), len(axes_to_check)):
                axes_to_check[j].set_visible(False)

            # ENHANCED: Add comprehensive legend for chiefdom plots
            legend_elements = [plt.Line2D([0], [0], color=color, lw=2, label=label, marker='o', markersize=3)
                             for variable, color, label in zip(variables, colors, labels)]
            fig.legend(handles=legend_elements, loc='center right',
                      bbox_to_anchor=(0.98, 0.5), ncol=1, fontsize=6,
                      frameon=True, fancybox=True, shadow=True,
                      framealpha=0.9, facecolor='white', edgecolor='gray')

            # Add common labels
            fig.text(0.5, 0.02, 'Year', ha='center', va='center', fontsize=12, fontweight='bold')
            fig.text(0.02, 0.5, y_axis_label, ha='center', va='center',
                     rotation='vertical', fontsize=12, fontweight='bold')

            plt.tight_layout(rect=[0.03, 0.03, 0.85, 0.96])
            pdf.savefig(fig, bbox_inches='tight')

            # Save as individual PNG
            safe_district_name = "".join(c for c in district if c.isalnum() or c in (' ', '-', '_')).strip().replace(' ', '_')
            filename = f'malaria_05_{district_idx:02d}_{safe_district_name}_chiefdoms'
            title = f'{main_title} - {district} (Chiefdoms)'
            save_and_track_figure(fig, filename, title, 'combined', 'chiefdoms')
            plt.close(fig)

    print(f"✅ PDF created: malaria_testing_analysis.pdf")

    # ==========================================
    # CREATE 3 SEPARATE HTML REPORTS
    # ==========================================
    print(f"\n🌐 Creating 3 separate HTML reports...")

    # Group images by section type
    sections = {
        'national': [],
        'comparison': [],
        'districts_overview': [],
        'individual_district': [],
        'chiefdoms': []
    }

    for img in html_images:
        section_type = img.get('section_type', 'combined')
        if section_type in sections:
            sections[section_type].append(img)

    def create_html_report(filename, title, description, include_comparison=True, test_type="all"):
        """Create HTML report with optional filtering by test type"""

        html_content = f'''
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>{title}</title>
        <style>
            body {{
                font-family: 'Arial', 'Helvetica', sans-serif;
                margin: 0;
                padding: 20px;
                background-color: #f4f6f9;
                line-height: 1.6;
                color: #2c3e50;
            }}
            .container {{
                max-width: 1200px;
                margin: 0 auto;
                background: white;
                padding: 30px;
                border-radius: 8px;
                box-shadow: 0 2px 10px rgba(0,0,0,0.1);
                border: 1px solid #e3e8ed;
            }}
            h1 {{
                color: #1e3a8a;
                text-align: center;
                border-bottom: 3px solid #2563eb;
                padding-bottom: 20px;
                margin-bottom: 40px;
                font-size: 2.2em;
                font-weight: 600;
            }}
            h2 {{
                color: white;
                background-color: #2563eb;
                margin: 40px -30px 30px -30px;
                padding: 20px 30px;
                font-size: 1.6em;
                font-weight: 500;
                border-left: 5px solid #1d4ed8;
                scroll-margin-top: 80px;
            }}
            h3 {{
                color: #1e3a8a;
                font-size: 1.3em;
                font-weight: 600;
                margin-top: 30px;
                margin-bottom: 15px;
                border-bottom: 2px solid #bfdbfe;
                padding-bottom: 8px;
                scroll-margin-top: 80px;
            }}
            .image-container {{
                background-color: #fafbfc;
                border: 1px solid #e3e8ed;
                border-radius: 6px;
                margin: 25px 0;
                padding: 20px;
                text-align: center;
            }}
            .image-container img {{
                max-width: 100%;
                height: auto;
                border: 1px solid #d1d9e0;
                border-radius: 4px;
            }}
            .image-title {{
                font-weight: 600;
                margin-bottom: 15px;
                color: #1e3a8a;
                font-size: 1.1em;
                background-color: #eff6ff;
                padding: 12px;
                border-radius: 4px;
                border-left: 4px solid #2563eb;
            }}
            .legend-notice {{
                background-color: #dcfce7;
                border: 2px solid #16a34a;
                border-radius: 6px;
                padding: 20px;
                margin: 20px 0;
                color: #15803d;
                font-weight: 500;
            }}
            .scaling-notice {{
                background-color: #fef3c7;
                border: 2px solid #f59e0b;
                border-radius: 6px;
                padding: 20px;
                margin: 20px 0;
                color: #92400e;
                font-weight: 500;
            }}

            /* Navigation Menu Styles */
            .section-nav {{
                position: fixed;
                top: 20px;
                right: 20px;
                background: white;
                border: 2px solid #2563eb;
                border-radius: 8px;
                max-width: 280px;
                min-width: 50px;
                z-index: 1000;
                box-shadow: 0 4px 20px rgba(0,0,0,0.15);
                transition: all 0.3s ease;
                overflow: hidden;
            }}

            .nav-header {{
                background: linear-gradient(135deg, #2563eb, #1d4ed8);
                color: white;
                padding: 12px 15px;
                cursor: pointer;
                display: flex;
                justify-content: space-between;
                align-items: center;
                font-weight: 600;
                font-size: 14px;
                user-select: none;
            }}

            .nav-toggle {{
                font-size: 18px;
                transition: transform 0.3s ease;
                line-height: 1;
            }}

            .nav-toggle.expanded {{
                transform: rotate(180deg);
            }}

            .nav-content {{
                max-height: 0;
                overflow: hidden;
                transition: max-height 0.3s ease;
                background: white;
            }}

            .nav-content.expanded {{
                max-height: 600px;
                padding: 10px 0;
            }}

            .nav-main-item {{
                display: block;
                color: #2563eb;
                text-decoration: none;
                padding: 8px 15px;
                font-size: 13px;
                font-weight: 600;
                border-bottom: 1px solid #f1f5f9;
                transition: all 0.2s ease;
                cursor: pointer;
                user-select: none;
            }}

            .nav-main-item:hover {{
                background-color: #eff6ff;
                color: #1d4ed8;
            }}

            .nav-main-item.active {{
                background-color: #dbeafe;
                color: #1d4ed8;
                border-left: 4px solid #2563eb;
            }}

            .scroll-progress {{
                position: fixed;
                top: 0;
                left: 0;
                width: 0%;
                height: 3px;
                background: linear-gradient(90deg, #2563eb, #60a5fa);
                z-index: 1001;
                transition: width 0.3s ease;
            }}

            @media (max-width: 768px) {{
                .section-nav {{
                    top: 10px;
                    right: 10px;
                    max-width: 250px;
                }}
                .container {{
                    padding: 15px;
                    margin: 0 10px;
                }}
            }}
        </style>
    </head>
    <body>
        <div class="scroll-progress" id="scrollProgress"></div>

        <div class="section-nav" id="sectionNav">
            <div class="nav-header" onclick="toggleNav()">
                <span class="nav-title">Navigation</span>
                <span class="nav-toggle" id="navToggle">▼</span>
            </div>
            <div class="nav-content" id="navContent">
                <a href="#national" class="nav-main-item" onclick="scrollToSection('national')">National Trends</a>
        '''

        if include_comparison and test_type == "all":
            html_content += '<a href="#comparison" class="nav-main-item" onclick="scrollToSection(\'comparison\')">Microscopy vs RDT</a>\n'

        html_content += f'''
                <a href="#districts-overview" class="nav-main-item" onclick="scrollToSection('districts-overview')">Districts Overview</a>
                <a href="#individual-districts" class="nav-main-item" onclick="scrollToSection('individual-districts')">Individual Districts</a>
                <a href="#chiefdoms" class="nav-main-item" onclick="scrollToSection('chiefdoms')">Chiefdoms</a>
            </div>
        </div>

        <div class="container">
            <h1>{title}<br><small>{description}</small></h1>

        <div id="analysis" class="section">
            <h2>Overview</h2>

            <div class="legend-notice">
                <strong>📊 ENHANCED VERSION:</strong> All plots now include comprehensive legends!
                <br><strong>Legend Features:</strong> Enhanced styling, clear labels, and consistent positioning across all visualizations.
            </div>

            <div class="scaling-notice">
        '''

        if test_type == "microscopy":
            html_content += '''
                <strong>Testing Method:</strong> Microscopy only
                <br><strong>Results:</strong> Positive (Pos) and Negative (Neg) results
                <br><strong>Age Groups:</strong> Under 5 (U5), 5-14 years, Over 15 years (O15)
                <br><strong>Setting:</strong> Health Facility (HF) only
                <br><strong>Individual Scaling:</strong> Each plot uses optimized scaling based on its specific data values.
            '''
        elif test_type == "rdt":
            html_content += '''
                <strong>Testing Method:</strong> Rapid Diagnostic Test (RDT) only
                <br><strong>Results:</strong> Positive (Pos) and Negative (Neg) results
                <br><strong>Age Groups:</strong> Under 5 (U5), 5-14 years, Over 15 years (O15)
                <br><strong>Setting:</strong> Health Facility (HF) only
                <br><strong>Individual Scaling:</strong> Each plot uses optimized scaling based on its specific data values.
            '''
        else:
            html_content += '''
                <strong>Testing Methods:</strong> Microscopy (Mic) and Rapid Diagnostic Test (RDT)
                <br><strong>Results:</strong> Positive (Pos) and Negative (Neg) results
                <br><strong>Age Groups:</strong> Under 5 (U5), 5-14 years, Over 15 years (O15)
                <br><strong>Setting:</strong> Health Facility (HF) only
                <br><strong>Individual Scaling:</strong> Each plot uses optimized scaling based on its specific data values.
            '''

        html_content += '''
            </div>
        '''

        # National Trends Section
        if sections['national']:
            html_content += f'''
                <div id="national" class="subsection">
                    <h3>National Trends</h3>
            '''
            for img in sections['national']:
                html_content += f'''
                    <div class="image-container">
                        <div class="image-title">{img['title']}</div>
                        <img src="data:image/png;base64,{img['image']}" alt="{img['title']}">
                    </div>
                '''
            html_content += '</div>\n'

        # Comparison Section (only for combined report)
        if include_comparison and test_type == "all" and sections['comparison']:
            html_content += f'''
                <div id="comparison" class="subsection">
                    <h3>Microscopy vs RDT Comparison</h3>
            '''
            for img in sections['comparison']:
                html_content += f'''
                    <div class="image-container">
                        <div class="image-title">{img['title']}</div>
                        <img src="data:image/png;base64,{img['image']}" alt="{img['title']}">
                    </div>
                '''
            html_content += '</div>\n'

        # Districts Overview Section
        if sections['districts_overview']:
            html_content += f'''
                <div id="districts-overview" class="subsection">
                    <h3>Districts Overview</h3>
            '''
            for img in sections['districts_overview']:
                html_content += f'''
                    <div class="image-container">
                        <div class="image-title">{img['title']}</div>
                        <img src="data:image/png;base64,{img['image']}" alt="{img['title']}">
                    </div>
                '''
            html_content += '</div>\n'

        # Individual Districts Section
        if sections['individual_district']:
            html_content += f'''
                <div id="individual-districts" class="subsection">
                    <h3>Individual Districts</h3>
            '''
            sorted_districts = sorted(sections['individual_district'], key=lambda x: x.get('filename', ''))
            for img in sorted_districts:
                html_content += f'''
                    <div class="image-container">
                        <div class="image-title">{img['title']}</div>
                        <img src="data:image/png;base64,{img['image']}" alt="{img['title']}">
                    </div>
                '''
            html_content += '</div>\n'

        # Chiefdoms Section
        if sections['chiefdoms']:
            html_content += f'''
                <div id="chiefdoms" class="subsection">
                    <h3>Chiefdoms</h3>
            '''
            sorted_chiefdoms = sorted(sections['chiefdoms'], key=lambda x: x.get('filename', ''))
            for img in sorted_chiefdoms:
                html_content += f'''
                    <div class="image-container">
                        <div class="image-title">{img['title']}</div>
                        <img src="data:image/png;base64,{img['image']}" alt="{img['title']}">
                    </div>
                '''
            html_content += '</div>\n'

        html_content += '</div>\n'  # Close analysis section

        # Add summary box
        if test_type == "microscopy":
            summary_text = '''
                <p><strong>Variables Analyzed:</strong> 6 microscopy variables (Positive & Negative, 3 age groups)</p>
                <p><strong>Testing Method:</strong> Microscopy only</p>
                <p><strong>Age Groups:</strong> Under 5, 5-14 years, Over 15 years</p>
                <p><strong>Setting:</strong> Health Facility only</p>
                <p><strong>Legend Enhancement:</strong> All plots include comprehensive legends with enhanced styling</p>
            '''
        elif test_type == "rdt":
            summary_text = '''
                <p><strong>Variables Analyzed:</strong> 6 RDT variables (Positive & Negative, 3 age groups)</p>
                <p><strong>Testing Method:</strong> Rapid Diagnostic Test (RDT) only</p>
                <p><strong>Age Groups:</strong> Under 5, 5-14 years, Over 15 years</p>
                <p><strong>Setting:</strong> Health Facility only</p>
                <p><strong>Legend Enhancement:</strong> All plots include comprehensive legends with enhanced styling</p>
            '''
        else:
            summary_text = '''
                <p><strong>Variables Analyzed:</strong> 12 variables (Microscopy & RDT, Positive & Negative, 3 age groups)</p>
                <p><strong>Testing Methods:</strong> Microscopy and Rapid Diagnostic Test (RDT)</p>
                <p><strong>Age Groups:</strong> Under 5, 5-14 years, Over 15 years</p>
                <p><strong>Setting:</strong> Health Facility only</p>
                <p><strong>Legend Enhancement:</strong> All plots include comprehensive legends with enhanced styling</p>
            '''

        html_content += f'''
                <div class="stats-box" style="background-color: #eff6ff; border: 2px solid #2563eb; border-radius: 6px; padding: 20px; margin-top: 50px; text-align: center; color: #1e3a8a;">
                    <h3>Malaria Testing Analysis Summary - Enhanced with Complete Legends</h3>
                    {summary_text}
                </div>
            </div>

            <script>
                let navExpanded = true;

                function toggleNav() {{
                    const nav = document.getElementById('sectionNav');
                    const content = document.getElementById('navContent');
                    const toggle = document.getElementById('navToggle');

                    navExpanded = !navExpanded;

                    if (navExpanded) {{
                        content.classList.add('expanded');
                        toggle.classList.add('expanded');
                    }} else {{
                        content.classList.remove('expanded');
                        toggle.classList.remove('expanded');
                    }}
                }}

                function scrollToSection(sectionId) {{
                    const target = document.getElementById(sectionId);
                    if (target) {{
                        target.scrollIntoView({{
                            behavior: 'smooth',
                            block: 'start'
                        }});
                        updateActiveNavItem(sectionId);
                    }}
                }}

                function updateActiveNavItem(sectionId) {{
                    document.querySelectorAll('.nav-main-item').forEach(item => {{
                        item.classList.remove('active');
                    }});

                    const activeItem = document.querySelector(`a[href="#${{sectionId}}"]`);
                    if (activeItem) {{
                        activeItem.classList.add('active');
                    }}
                }}

                function updateScrollProgress() {{
                    const scrollTop = window.pageYOffset;
                    const docHeight = document.body.scrollHeight - window.innerHeight;
                    const scrollPercent = (scrollTop / docHeight) * 100;
                    document.getElementById('scrollProgress').style.width = scrollPercent + '%';
                }}

                window.addEventListener('scroll', updateScrollProgress);
                document.addEventListener('DOMContentLoaded', updateScrollProgress);

                document.querySelectorAll('a[href^="#"]').forEach(anchor => {{
                    anchor.addEventListener('click', function (e) {{
                        e.preventDefault();
                        const targetId = this.getAttribute('href').substring(1);
                        scrollToSection(targetId);
                    }});
                }});

                document.addEventListener('DOMContentLoaded', function() {{
                    if (window.location.hash) {{
                        const hash = window.location.hash.substring(1);
                        scrollToSection(hash);
                    }}
                }});
            </script>
        </body>
        </html>
        '''

        return html_content

    # Create the 3 HTML reports
    html_files = []

    # 1. Microscopy Only Report
    mic_filename = f'{output_base_dir}/malaria_testing_microscopy_report.html'
    mic_html = create_html_report(
        mic_filename,
        "Malaria Microscopy Testing Analysis - Enhanced",
        "Microscopy Testing Results by Age Groups (2021-2024) - Complete Legends",
        include_comparison=False,
        test_type="microscopy"
    )
    with open(mic_filename, 'w', encoding='utf-8') as f:
        f.write(mic_html)
    html_files.append(mic_filename)
    print(f"✅ Enhanced Microscopy HTML report created: {mic_filename}")

    # 2. RDT Only Report
    rdt_filename = f'{output_base_dir}/malaria_testing_rdt_report.html'
    rdt_html = create_html_report(
        rdt_filename,
        "Malaria RDT Testing Analysis - Enhanced",
        "RDT Testing Results by Age Groups (2021-2024) - Complete Legends",
        include_comparison=False,
        test_type="rdt"
    )
    with open(rdt_filename, 'w', encoding='utf-8') as f:
        f.write(rdt_html)
    html_files.append(rdt_filename)
    print(f"✅ Enhanced RDT HTML report created: {rdt_filename}")

    # 3. Combined Report
    combined_filename = f'{output_base_dir}/malaria_testing_combined_report.html'
    combined_html = create_html_report(
        combined_filename,
        "Malaria Testing Analysis - Combined Enhanced",
        "Microscopy and RDT Testing Results by Age Groups (2021-2024) - Complete Legends",
        include_comparison=True,
        test_type="all"
    )
    with open(combined_filename, 'w', encoding='utf-8') as f:
        f.write(combined_html)
    html_files.append(combined_filename)
    print(f"✅ Enhanced Combined HTML report created: {combined_filename}")

    # ==========================================
    # CREATE ZIP PACKAGE
    # ==========================================
    print(f"\n📦 Creating comprehensive ZIP package...")

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    zip_filename = f'{output_base_dir}/malaria_testing_analysis_enhanced_{timestamp}.zip'

    with zipfile.ZipFile(zip_filename, 'w', zipfile.ZIP_DEFLATED) as zipf:
        # Add PNG files
        for png_file in all_png_files:
            zipf.write(png_file, f"png_files/{Path(png_file).name}")

        # Add PDF file
        pdf_file = f'{output_base_dir}/pdf_files/malaria_testing_analysis.pdf'
        if os.path.exists(pdf_file):
            zipf.write(pdf_file, 'malaria_testing_analysis.pdf')

        # Add all HTML files to ZIP
        for html_file in html_files:
            zipf.write(html_file, Path(html_file).name)

    print(f"✅ Enhanced ZIP package created: {zip_filename}")

    # ==========================================
    # FINAL SUMMARY
    # ==========================================
    print("\n" + "=" * 70)
    print("🎉 ENHANCED MALARIA TESTING ANALYSIS COMPLETE!")
    print("=" * 70)

    print(f"📊 **Variables Analyzed:**")
    for i, (var, label) in enumerate(zip(variables, labels)):
        print(f"   • {var} → {label}")

    print(f"\n📈 **Analysis Structure with Complete Legends:**")
    print(f"   ✅ National trends (all testing variables) - LEGEND ADDED")
    print(f"   ✅ Microscopy vs RDT comparison - LEGENDS ADDED")
    print(f"   ✅ District overview subplots - LEGEND ADDED")
    print(f"   ✅ Individual district analysis - LEGENDS ADDED")
    print(f"   ✅ Chiefdom subplots by district - LEGENDS ADDED")

    print(f"\n📁 **Enhanced Files Generated:**")
    print(f"   • {len(all_png_files)} PNG files (all with legends)")
    print(f"   • 1 comprehensive PDF (all plots with legends)")
    print(f"   • 3 enhanced HTML reports (Microscopy, RDT, Combined)")
    print(f"   • 1 enhanced ZIP package")

    print(f"\n🏆 **Legend Enhancement Features:**")
    print(f"   • Enhanced styling with shadows and borders")
    print(f"   • Bold text for better readability")
    print(f"   • Consistent positioning across all plots")
    print(f"   • Proper spacing and formatting")
    print(f"   • Markers included in legend elements")

    print(f"\n🔬 **Testing Methods Covered:**")
    print(f"   • Microscopy testing (positive/negative)")
    print(f"   • RDT testing (positive/negative)")
    print(f"   • All age groups (Under 5, 5-14, Over 15)")
    print(f"   • Health facility setting only")

    print(f"\n✨ **Enhanced malaria testing analysis ready with complete legends!**")

    return df

# Run the enhanced malaria testing analysis
if __name__ == "__main__":
    # Customize these parameters as needed:
    INPUT_FILE = "/content/comparative_analysis.xlsx"
    OUTPUT_DIR = 'malaria_testing_plots_enhanced'
    MAIN_TITLE = "Malaria Testing Results"
    Y_AXIS_LABEL = "Number of tests"

    # Run the function
    df = create_malaria_testing_analysis(
        input_file=INPUT_FILE,
        output_base_dir=OUTPUT_DIR,
        main_title=MAIN_TITLE,
        y_axis_label=Y_AXIS_LABEL
    )

    if df is not None:
        print(f"\nDataframe loaded successfully: {df.shape}")
        print("Available columns:", df.columns.tolist())
```
