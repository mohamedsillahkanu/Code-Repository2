"""
Professional Malaria Trend Maps Generator
Creates comprehensive trend visualization maps with basic HTML output - EXACT format as sample
Variables: crude_incidence, adjusted1, adjusted2, adjusted3
Execute this file directly in your Python environment
"""

def create_comprehensive_malaria_trend_maps(output_dir='malaria_trend_maps/'):
    import os
    import matplotlib.pyplot as plt
    import pandas as pd
    import numpy as np
    import geopandas as gpd
    import re
    from matplotlib.colors import BoundaryNorm
    from matplotlib.cm import RdBu_r
    import matplotlib.patches as mpatches
    import zipfile
    from pathlib import Path
    import math
    import base64
    from io import BytesIO

    print("🚀 Starting Professional Malaria Trend Maps Generation...")
    print("=" * 60)

    # Create output directory if it doesn't exist
    os.makedirs(output_dir, exist_ok=True)

    # Create subdirectories for different map types
    national_dir = os.path.join(output_dir, 'malaria_trend_maps_national')
    district_dir = os.path.join(output_dir, 'malaria_trend_maps_district')
    chiefdom_dir = os.path.join(output_dir, 'malaria_trend_maps_chiefdom')
    subplot_dir = os.path.join(output_dir, 'malaria_trend_maps_subplots')

    for directory in [national_dir, district_dir, chiefdom_dir, subplot_dir]:
        os.makedirs(directory, exist_ok=True)

    print(f"📁 Created directory structure in: {output_dir}")

    # Variables to analyze - MALARIA SPECIFIC
    variables = ['crude_incidence', 'adjusted1', 'adjusted2', 'adjusted3']

    # Store all maps for HTML generation
    html_images = []
    all_map_files = []

    # Read data
    print("📊 Loading malaria data files...")
    try:
        df1 = pd.read_excel("/content/2024_snt_data.xlsx")
        shapefile = gpd.read_file("/content/Chiefdom2021.shp")
        print(f"✓ Data loaded successfully")
        print(f"   - Excel data: {len(df1)} rows")
        print(f"   - Shapefile: {len(shapefile)} features")
    except FileNotFoundError as e:
        print(f"❌ Error loading files: {e}")
        print("Please ensure files exist at:")
        print("   - /content/2024_snt_data.xlsx")
        print("   - /content/Chiefdom2021.shp")
        return

    # Process each variable
    for variable in variables:
        print(f"\n{'='*60}")
        print(f"🗺️  PROCESSING VARIABLE: {variable.upper()}")
        print(f"{'='*60}")

        # Identify year columns for 2021-2024
        pattern = re.compile(f'^{variable}_(\d{{4}})$')
        year_cols = [col for col in df1.columns
                     if pattern.match(col) and 2021 <= int(pattern.match(col).group(1)) <= 2024]

        print(f"📈 Found {len(year_cols)} year columns: {year_cols}")

        if len(year_cols) < 2:
            print("❌ Error: Need at least 2 years of data to calculate percentage change")
            continue

        # Calculate overall percentage change for each chiefdom (2021 to 2024 specifically)
        def calculate_overall_change(row):
            value_2021 = None
            value_2024 = None

            for col in year_cols:
                year = int(re.search(r'(\d{4})', col).group(1))
                if year == 2021 and pd.notna(row[col]):
                    value_2021 = row[col]
                elif year == 2024 and pd.notna(row[col]):
                    value_2024 = row[col]

            if value_2021 is not None and value_2024 is not None and value_2021 > 0:
                pct_change = ((value_2024 - value_2021) / value_2021) * 100
                return pct_change, 2021, 2024, value_2021, value_2024

            return np.nan, np.nan, np.nan, np.nan, np.nan

        # Calculate percentage changes
        print("📊 Calculating percentage changes...")
        df1[['overall_pct_change', 'first_year', 'last_year', 'first_value', 'last_value']] = df1.apply(
            lambda row: pd.Series(calculate_overall_change(row)), axis=1
        )

        # Merge with shapefile
        gdf = shapefile.merge(df1, on=['FIRST_DNAM', 'FIRST_CHIE'], how='left', validate='1:1')

        # Fixed bins for percentage change (-70% to >+20%, 10 bins)
        def create_fixed_bins():
            bins = np.linspace(-70, 20, 11)  # 11 values create 10 bins
            return bins

        def create_map_with_legend(gdf_plot, title, filename, output_directory, stats_text=None,
                                  show_dnam_boundaries=False, filter_to_data=False, show_names=False,
                                  name_column=None, use_simple_colors=False):
            """Helper function to create standardized maps"""
            fig, ax = plt.subplots(1, 1, figsize=(14, 10))

            # Determine what to show as base
            if filter_to_data and len(gdf_plot) > 0:
                if show_dnam_boundaries:
                    dnam_values = gdf_plot['FIRST_DNAM'].dropna().unique()
                    base_gdf = gdf[gdf['FIRST_DNAM'].isin(dnam_values)]
                else:
                    base_gdf = gdf_plot
                base_gdf.boundary.plot(ax=ax, color='lightgray', linewidth=0.5, alpha=0.3)
            else:
                gdf.boundary.plot(ax=ax, color='lightgray', linewidth=0.5, alpha=0.3)

            if len(gdf_plot) > 0:
                valid_data = gdf_plot.dropna(subset=['overall_pct_change'])

                if len(valid_data) > 0:
                    if use_simple_colors:
                        colors = ['#47B5FF' if x < 0 else 'pink' for x in valid_data['overall_pct_change']]

                        valid_data.plot(
                            color=colors,
                            ax=ax,
                            legend=False,
                            edgecolor='gray',
                            linewidth=1.0
                        )

                        legend_elements = [
                            mpatches.Patch(color='#47B5FF', label='Decreasing Incidence'),
                            mpatches.Patch(color='pink', label='Increasing Incidence')
                        ]

                        legend = ax.legend(handles=legend_elements, loc='center left', bbox_to_anchor=(1, 0.5),
                                          title='Trend Direction',
                                          title_fontsize=12, fontsize=10)
                        legend.get_title().set_fontweight('bold')
                    else:
                        bins = create_fixed_bins()

                        bin_labels = []
                        for i in range(len(bins) - 1):
                            if i == len(bins) - 2:
                                bin_labels.append(f"{bins[i]:+.0f}% to >+{bins[i+1]:.0f}%")
                            else:
                                bin_labels.append(f"{bins[i]:+.0f}% to {bins[i+1]:+.0f}%")

                        cmap = RdBu_r
                        norm = BoundaryNorm(bins, cmap.N)

                        valid_data.plot(
                            column='overall_pct_change',
                            ax=ax,
                            cmap=cmap,
                            norm=norm,
                            legend=False,
                            edgecolor='gray',
                            linewidth=1.0
                        )

                        legend_elements = []
                        for i, (bin_label, color_val) in enumerate(zip(bin_labels, np.linspace(0, 1, len(bin_labels)))):
                            color = cmap(color_val)
                            legend_elements.append(mpatches.Patch(color=color, label=bin_label))

                        legend = ax.legend(handles=legend_elements, loc='center left', bbox_to_anchor=(1, 0.5),
                                          title='Overall % Change\n(Red=Increase, Blue=Decrease)',
                                          title_fontsize=12, fontsize=10)
                        legend.get_title().set_fontweight('bold')

            # Add FIRST_DNAM boundaries if requested
            if show_dnam_boundaries:
                if filter_to_data and len(gdf_plot) > 0:
                    dnam_values = gdf_plot['FIRST_DNAM'].dropna().unique()
                    dnam_boundaries_data = gdf[gdf['FIRST_DNAM'].isin(dnam_values)]
                else:
                    dnam_boundaries_data = gdf

                for dnam in dnam_boundaries_data['FIRST_DNAM'].dropna().unique():
                    dnam_geom = dnam_boundaries_data[dnam_boundaries_data['FIRST_DNAM'] == dnam]
                    dnam_dissolved = dnam_geom.dissolve(by='FIRST_DNAM')
                    dnam_dissolved.boundary.plot(ax=ax, color='black', linewidth=2.5, alpha=1.0)

            # Add names if requested
            if show_names and name_column and len(gdf_plot) > 0:
                if name_column == 'FIRST_DNAM':
                    dnam_groups = gdf_plot.groupby('FIRST_DNAM')
                    for dnam_name, group in dnam_groups:
                        union_geom = group.geometry.unary_union
                        if hasattr(union_geom, 'centroid'):
                            centroid = union_geom.centroid
                            ax.annotate(dnam_name,
                                       xy=(centroid.x, centroid.y),
                                       xytext=(3, 3), textcoords="offset points",
                                       fontsize=10, fontweight='bold',
                                       bbox=dict(boxstyle='round,pad=0.3', facecolor='white', alpha=0.8),
                                       ha='center')

            ax.set_title(title, fontsize=16, fontweight='bold', pad=20)
            ax.set_axis_off()

            if stats_text:
                ax.text(0.02, 0.98, stats_text, transform=ax.transAxes, fontsize=11,
                        verticalalignment='top', bbox=dict(boxstyle='round', facecolor='white', alpha=0.8))

            map_path = os.path.join(output_directory, filename)
            plt.tight_layout()
            plt.savefig(map_path, dpi=400, bbox_inches='tight')
            plt.close()
            print(f"   ✓ Saved: {filename}")
            all_map_files.append(map_path)

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
                'filename': filename
            })

            return map_path

        def create_subplot_map(first_dnam, gdf_valid, start_year, end_year):
            """Create professional subplot showing all FIRST_CHIE under a FIRST_DNAM"""
            gdf_dnam = gdf_valid[gdf_valid['FIRST_DNAM'] == first_dnam].copy()

            if len(gdf_dnam) == 0:
                return None

            # Get unique chiefdoms and sort them
            chiefdoms = sorted(gdf_dnam['FIRST_CHIE'].dropna().unique())
            n_chiefdoms = len(chiefdoms)

            if n_chiefdoms == 0:
                return None

            # Calculate optimal grid dimensions (4 columns max, balanced layout)
            n_cols = min(4, n_chiefdoms)
            n_rows = math.ceil(n_chiefdoms / n_cols)

            # Create figure with professional styling
            fig = plt.figure(figsize=(4*n_cols, 3.5*n_rows))
            fig.patch.set_facecolor('white')

            # Create main title
            var_title = variable.replace('_', ' ').title()
            main_title = f"{var_title} Change by Chiefdom/Zone in : {first_dnam.upper()} District ({start_year} to {end_year})"
            subtitle = "Pink = Increasing | Blue = Decreasing"

            # Add main title and subtitle
            fig.suptitle(main_title, fontsize=16, fontweight='bold', y=0.95)
            fig.text(0.5, 0.92, subtitle, ha='center', fontsize=12, style='italic')

            # Create subplots with proper spacing
            for i, chiefdom in enumerate(chiefdoms):
                # Create subplot
                ax = plt.subplot(n_rows, n_cols, i + 1)

                # Filter data for this chiefdom
                gdf_chie = gdf_dnam[gdf_dnam['FIRST_CHIE'] == chiefdom].copy()

                if len(gdf_chie) > 0:
                    # Get percentage change
                    pct_change = gdf_chie['overall_pct_change'].iloc[0]

                    # Set colors - professional blue/pink scheme
                    if pct_change < 0:
                        color = '#4A90E2'  # Professional blue
                        text_color = 'white'
                    elif pct_change > 0:
                        color = '#FF69B4'  # Professional pink
                        text_color = 'white'
                    else:
                        color = '#E8E8E8'  # Light gray for stable
                        text_color = 'black'

                    # Plot the chiefdom with clean styling
                    gdf_chie.plot(
                        color=color,
                        ax=ax,
                        edgecolor='black',
                        linewidth=1.2
                    )

                    # Add percentage text in center of polygon
                    bounds = gdf_chie.bounds
                    center_x = (bounds.minx.iloc[0] + bounds.maxx.iloc[0]) / 2
                    center_y = (bounds.miny.iloc[0] + bounds.maxy.iloc[0]) / 2

                    # Add percentage text with background
                    ax.text(center_x, center_y, f'{pct_change:+.1f}%',
                           ha='center', va='center', fontsize=14, fontweight='bold',
                           color=text_color,
                           bbox=dict(boxstyle='round,pad=0.3', facecolor=color,
                                    edgecolor='black', alpha=0.8))

                    # Add percentage in title
                    title_with_pct = f'{chiefdom.upper()}\n{pct_change:+.1f}%'
                    ax.set_title(title_with_pct, fontsize=10, fontweight='bold', pad=8)

                # Clean up axes
                ax.set_axis_off()
                ax.set_aspect('equal')

            plt.tight_layout(rect=[0, 0.08, 1, 0.90])

            # Save with high quality
            safe_dnam_name = "".join(c for c in first_dnam if c.isalnum() or c in (' ', '-', '_')).rstrip()
            filename = f'{variable}_subplot_all_chiefdoms_{safe_dnam_name}.png'
            map_path = os.path.join(subplot_dir, filename)

            plt.savefig(map_path, dpi=400, bbox_inches='tight',
                       facecolor='white', edgecolor='black')
            plt.close()
            print(f"   ✓ Saved: {filename}")
            all_map_files.append(map_path)

            # Convert to base64 for HTML
            buffer = BytesIO()
            fig.savefig(buffer, format='png', dpi=150, bbox_inches='tight')
            buffer.seek(0)
            image_base64 = base64.b64encode(buffer.getvalue()).decode('utf-8')
            buffer.close()

            html_images.append({
                'title': f'{var_title} - {first_dnam} Chiefdoms',
                'image': image_base64,
                'variable': variable,
                'filename': filename
            })

            return map_path

        # MAIN PROCESSING STARTS HERE
        # Remove rows with no percentage change data
        gdf_valid = gdf.dropna(subset=['overall_pct_change']).copy()

        if len(gdf_valid) == 0:
            print("❌ Error: No valid percentage change data found")
            continue

        print(f"✓ Processing {len(gdf_valid)} valid chiefdoms with percentage change data")

        # Calculate national-level change
        averages = df1[year_cols].mean(axis=0)
        avg_df = averages.reset_index()
        avg_df.columns = ['Year', f'National_{variable}']
        avg_df['Year'] = avg_df['Year'].str.extract(r'(\d{4})').astype(int)
        avg_df = avg_df.sort_values('Year').reset_index(drop=True)

        y_start = avg_df[f'National_{variable}'].iloc[0]
        y_end = avg_df[f'National_{variable}'].iloc[-1]
        national_overall_change = ((y_end - y_start) / y_start) * 100 if y_start > 0 else 0
        start_year = avg_df['Year'].iloc[0]
        end_year = avg_df['Year'].iloc[-1]

        print(f"📊 Analysis period: {start_year} to {end_year}")
        print(f"📈 National change: {national_overall_change:+.1f}%")

        # 1. NATIONAL OVERVIEW MAP
        print(f"\n🗺️  Creating National Overview Map for {variable}...")
        total_chiefdoms = len(gdf_valid)
        national_increasing = len(gdf_valid[gdf_valid['overall_pct_change'] > 0])
        national_decreasing = len(gdf_valid[gdf_valid['overall_pct_change'] < 0])
        national_stable = len(gdf_valid[gdf_valid['overall_pct_change'] == 0])

        var_title = variable.replace('_', ' ').title()
        national_title = f"National {var_title} Change by Chiefdom/Zone ({start_year} to {end_year})\nNational Change: {national_overall_change:+.1f}% | Increasing ({national_increasing}) | Decreasing ({national_decreasing}) | Stable ({national_stable})"

        create_map_with_legend(
            gdf_valid,
            national_title,
            f'{variable}_national_change_map.png',
            national_dir,
            show_dnam_boundaries=True,
            filter_to_data=True,
            show_names=True,
            name_column='FIRST_DNAM',
            use_simple_colors=True
        )

        # 2. INDIVIDUAL MAPS FOR EACH FIRST_DNAM
        first_dnam_values = gdf_valid['FIRST_DNAM'].dropna().unique()
        print(f"\n🏘️  Creating Individual District Maps for {variable} ({len(first_dnam_values)} districts)...")

        for i, first_dnam in enumerate(first_dnam_values, 1):
            print(f"   [{i}/{len(first_dnam_values)}] Processing {first_dnam}...")
            gdf_dnam = gdf_valid[gdf_valid['FIRST_DNAM'] == first_dnam].copy()

            if len(gdf_dnam) == 0:
                continue

            dnam_increasing = len(gdf_dnam[gdf_dnam['overall_pct_change'] > 0])
            dnam_decreasing = len(gdf_dnam[gdf_dnam['overall_pct_change'] < 0])
            dnam_stable = len(gdf_dnam[gdf_dnam['overall_pct_change'] == 0])

            safe_dnam_name = "".join(c for c in first_dnam if c.isalnum() or c in (' ', '-', '_')).rstrip()

            dnam_title = f"{var_title} Change - {first_dnam} ({start_year} to {end_year})\nIncreasing ({dnam_increasing}) | Decreasing ({dnam_decreasing}) | Stable ({dnam_stable})"

            create_map_with_legend(
                gdf_dnam,
                dnam_title,
                f'{variable}_{safe_dnam_name}_map.png',
                district_dir,
                filter_to_data=True,
                show_names=True,
                name_column='FIRST_CHIE',
                use_simple_colors=True
            )

        # 3. PROFESSIONAL SUBPLOT MAPS
        print(f"\n🎨 Creating Professional Subplot Maps for {variable} ({len(first_dnam_values)} districts)...")
        for i, first_dnam in enumerate(first_dnam_values, 1):
            print(f"   [{i}/{len(first_dnam_values)}] Creating subplot for {first_dnam}...")
            create_subplot_map(first_dnam, gdf_valid, start_year, end_year)

        plt.close('all')  # Clean up memory

    # ==========================================
    # CREATE HTML REPORT WITH COLLAPSIBLE NAVIGATION - EXACT FORMAT AS SAMPLE
    # ==========================================
    print(f"\n🌐 Creating HTML report with collapsible navigation - exact format as sample...")

    html_filename = os.path.join(output_dir, 'malaria_trend_maps_report.html')

    # Group images by variable
    grouped_images = {}
    for variable in variables:
        grouped_images[variable] = []

    for img in html_images:
        var = img['variable']
        if var in grouped_images:
            grouped_images[var].append(img)

    # HTML with collapsible navigation - EXACT format as your sample
    html_content = f"""
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Malaria Trend Maps Report - Enhanced Navigation (2021-2024)</title>
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
            }}
            h3 {{
                color: #1e3a8a;
                margin-top: 35px;
                margin-bottom: 20px;
                padding-bottom: 8px;
                border-bottom: 2px solid #e3e8ed;
                font-size: 1.3em;
                font-weight: 500;
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
            .toc {{
                background-color: #eff6ff;
                border: 2px solid #2563eb;
                border-radius: 6px;
                padding: 25px;
                margin-bottom: 40px;
            }}
            .toc h3 {{
                color: #1e3a8a;
                margin-top: 0;
                margin-bottom: 15px;
                border-bottom: 2px solid #bfdbfe;
                padding-bottom: 10px;
                font-size: 1.3em;
            }}
            .toc ul {{
                list-style-type: none;
                padding-left: 0;
                margin: 0;
            }}
            .toc li {{
                margin: 6px 0;
            }}
            .toc a {{
                color: #1e3a8a;
                text-decoration: none;
                padding: 8px 12px;
                display: inline-block;
                border-radius: 4px;
                transition: background-color 0.2s ease;
                font-weight: 500;
            }}
            .toc a:hover {{
                background-color: #dbeafe;
                color: #1d4ed8;
            }}
            .variable-section {{
                border-top: 2px solid #e3e8ed;
                margin-top: 50px;
                padding-top: 10px;
            }}
            .stats-box {{
                background-color: #eff6ff;
                border: 2px solid #2563eb;
                border-radius: 6px;
                padding: 20px;
                margin: 20px 0;
                text-align: center;
                color: #1e3a8a;
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

            /* Enhanced Navigation Menu Styles */
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
                max-height: 500px;
                padding: 10px 0;
            }}

            .section-nav.collapsed {{
                width: 50px;
            }}

            .section-nav.collapsed .nav-header {{
                justify-content: center;
            }}

            .section-nav.collapsed .nav-title {{
                display: none;
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

            .nav-main-item.has-submenu {{
                position: relative;
            }}

            .nav-main-item.has-submenu::after {{
                content: "▶";
                position: absolute;
                right: 15px;
                font-size: 10px;
                transition: transform 0.2s ease;
            }}

            .nav-main-item.has-submenu.expanded::after {{
                transform: rotate(90deg);
            }}

            .nav-submenu {{
                max-height: 0;
                overflow: hidden;
                transition: max-height 0.3s ease;
                background-color: #f8fafc;
                border-left: 3px solid #bfdbfe;
                margin-left: 15px;
                margin-right: 5px;
                border-radius: 0 4px 4px 0;
            }}

            .nav-submenu.expanded {{
                max-height: 200px;
                padding: 5px 0;
            }}

            .nav-sub-item {{
                display: block;
                color: #64748b;
                text-decoration: none;
                padding: 6px 15px;
                font-size: 12px;
                transition: all 0.2s ease;
                border-bottom: 1px solid #e2e8f0;
            }}

            .nav-sub-item:last-child {{
                border-bottom: none;
            }}

            .nav-sub-item:hover {{
                background-color: #e0f2fe;
                color: #0369a1;
                padding-left: 20px;
            }}

            .nav-sub-item.active {{
                background-color: #dbeafe;
                color: #1d4ed8;
                font-weight: 600;
                border-left: 3px solid #2563eb;
            }}

            /* Mobile responsiveness */
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

            /* Smooth scrolling indicator */
            .nav-main-item.active {{
                background-color: #dbeafe;
                color: #1d4ed8;
                border-left: 4px solid #2563eb;
            }}

            /* Scroll progress bar */
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
                <a href="#summary" class="nav-main-item" onclick="scrollToSection('summary')">📊 Summary</a>
    """

    # Add navigation with submenus for each variable
    for variable in variables:
        if variable in grouped_images and len(grouped_images[variable]) > 0:
            var_title = variable.replace('_', ' ').title()

            # Create main menu item with submenu
            html_content += f'''
                <div class="nav-main-item has-submenu" onclick="toggleSubmenu('{variable}')" id="nav-{variable}">
                    {var_title}
                </div>
                <div class="nav-submenu" id="submenu-{variable}">
                    <a href="#{variable}-national" class="nav-sub-item" onclick="scrollToSection('{variable}-national')">National Maps</a>
                    <a href="#{variable}-districts" class="nav-sub-item" onclick="scrollToSection('{variable}-districts')">District Maps</a>
                    <a href="#{variable}-subplots" class="nav-sub-item" onclick="scrollToSection('{variable}-subplots')">Subplot Maps</a>
                </div>
            '''

    html_content += """
            </div>
        </div>

        <div class="container">
            <h1>Malaria Trend Maps Report (2021-2024)<br><small>With Enhanced Collapsible Navigation</small></h1>

            <div class="scaling-notice">
                <strong>IMPORTANT:</strong> This report contains comprehensive malaria trend maps for all geographic levels.
                Maps use color coding to show trend directions: <strong>Blue = Decreasing incidence</strong>, <strong>Pink = Increasing incidence</strong>.
                <br><strong>Navigation:</strong> Use the collapsible menu on the right to jump between sections and map types.
            </div>

            <div class="toc">
                <h3>Table of Contents</h3>
                <ul>
                    <li><a href="#summary">Summary</a></li>
    """

    # Add TOC entries for each variable
    for variable in variables:
        if variable in grouped_images:
            var_title = variable.replace('_', ' ').title()
            html_content += f'                    <li><a href="#{variable}">{var_title}</a></li>\n'

    html_content += """
                </ul>
            </div>

            <div id="summary" class="variable-section">
                <h2>Summary</h2>
                <div class="stats-box">
                    <h3>Malaria Trend Maps Analysis Summary</h3>
                    <p><strong>Variables:</strong> Crude Incidence, Adjusted1, Adjusted2, Adjusted3</p>
                    <p><strong>Time Period:</strong> 2021-2024 (4 years)</p>
                    <p><strong>Geographic Levels:</strong> National → District → Chiefdom</p>
                    <p><strong>Color Scheme:</strong> Blue (decreasing), Pink (increasing)</p>
                </div>
            </div>
    """

    # Add each variable section with properly ordered maps
    for variable in variables:
        if variable not in grouped_images:
            continue

        var_title = variable.replace('_', ' ').title()
        html_content += f'''
            <div id="{variable}" class="variable-section">
                <h2>{var_title}</h2>

                <div class="scaling-notice">
                    <strong>Map Types:</strong> This section contains all maps for {var_title} analysis.
                    <br><strong>Color Coding:</strong>
                    <span style="color: blue; font-weight: bold;">Blue areas</span> (decreasing incidence),
                    <span style="color: #FF69B4; font-weight: bold;">Pink areas</span> (increasing incidence).
                </div>
        '''

        # Get all images for this variable and sort by filename
        var_images = grouped_images[variable]
        var_images_sorted = sorted(var_images, key=lambda x: x.get('filename', ''))

        # Organize maps by type to match navigation order for ALL variables
        national_maps = []
        district_maps = []
        subplot_maps = []

        for img in var_images_sorted:
            filename = img.get('filename', '')
            if 'national' in filename.lower():
                national_maps.append(img)
            elif 'subplot' in filename.lower():
                subplot_maps.append(img)
            else:
                district_maps.append(img)

        # 1. NATIONAL MAPS SECTION - FIRST FOR ALL VARIABLES
        if national_maps:
            html_content += f'''
                <div id="{variable}-national">
                    <h3>National {var_title} Maps</h3>
            '''
            for img in national_maps:
                html_content += f'''
                    <div class="image-container">
                        <div class="image-title">{img['title']}</div>
                        <img src="data:image/png;base64,{img['image']}" alt="{img['title']}">
                    </div>
                '''
            html_content += '</div>\n'

        # 2. DISTRICT MAPS SECTION - SECOND FOR ALL VARIABLES
        if district_maps:
            html_content += f'''
                <div id="{variable}-districts">
                    <h3>District {var_title} Maps</h3>
            '''
            for img in district_maps:
                html_content += f'''
                    <div class="image-container">
                        <div class="image-title">{img['title']}</div>
                        <img src="data:image/png;base64,{img['image']}" alt="{img['title']}">
                    </div>
                '''
            html_content += '</div>\n'

        # 3. SUBPLOT MAPS SECTION - THIRD FOR ALL VARIABLES
        if subplot_maps:
            html_content += f'''
                <div id="{variable}-subplots">
                    <h3>Chiefdom Subplot {var_title} Maps</h3>
            '''
            for img in subplot_maps:
                html_content += f'''
                    <div class="image-container">
                        <div class="image-title">{img['title']}</div>
                        <img src="data:image/png;base64,{img['image']}" alt="{img['title']}">
                    </div>
                '''
            html_content += '</div>\n'

        html_content += '</div>\n'

    html_content += '''
        </div>

        <script>
            // Navigation state management
            let navExpanded = true;
            let activeSubmenus = new Set();

            // Toggle main navigation
            function toggleNav() {
                const nav = document.getElementById('sectionNav');
                const content = document.getElementById('navContent');
                const toggle = document.getElementById('navToggle');

                navExpanded = !navExpanded;

                if (navExpanded) {
                    nav.classList.remove('collapsed');
                    content.classList.add('expanded');
                    toggle.classList.add('expanded');
                } else {
                    nav.classList.add('collapsed');
                    content.classList.remove('expanded');
                    toggle.classList.remove('expanded');
                    // Close all submenus when collapsing
                    activeSubmenus.clear();
                    document.querySelectorAll('.nav-submenu').forEach(submenu => {
                        submenu.classList.remove('expanded');
                    });
                    document.querySelectorAll('.nav-main-item.has-submenu').forEach(item => {
                        item.classList.remove('expanded');
                    });
                }
            }

            // Toggle submenu
            function toggleSubmenu(variable) {
                if (!navExpanded) return;

                const submenu = document.getElementById(`submenu-${variable}`);
                const mainItem = document.getElementById(`nav-${variable}`);

                if (activeSubmenus.has(variable)) {
                    activeSubmenus.delete(variable);
                    submenu.classList.remove('expanded');
                    mainItem.classList.remove('expanded');
                } else {
                    activeSubmenus.add(variable);
                    submenu.classList.add('expanded');
                    mainItem.classList.add('expanded');
                }
            }

            // Smooth scrolling for navigation links
            function scrollToSection(sectionId) {
                const target = document.getElementById(sectionId);
                if (target) {
                    target.scrollIntoView({
                        behavior: 'smooth',
                        block: 'start'
                    });

                    // Update active navigation item
                    updateActiveNavItem(sectionId);
                }
            }

            // Update active navigation item
            function updateActiveNavItem(sectionId) {
                // Remove active class from all items
                document.querySelectorAll('.nav-main-item, .nav-sub-item').forEach(item => {
                    item.classList.remove('active');
                });

                // Add active class to current section
                const activeItem = document.querySelector(`a[href="#${sectionId}"]`) ||
                                  document.getElementById(`nav-${sectionId}`);
                if (activeItem) {
                    activeItem.classList.add('active');
                }
            }

            // Scroll progress bar
            function updateScrollProgress() {
                const scrollTop = window.pageYOffset;
                const docHeight = document.body.scrollHeight - window.innerHeight;
                const scrollPercent = (scrollTop / docHeight) * 100;
                document.getElementById('scrollProgress').style.width = scrollPercent + '%';
            }

            // Section intersection observer for auto-highlighting
            function setupIntersectionObserver() {
                const sections = document.querySelectorAll('.variable-section');

                const observer = new IntersectionObserver((entries) => {
                    entries.forEach(entry => {
                        if (entry.isIntersecting) {
                            const sectionId = entry.target.id;
                            updateActiveNavItem(sectionId);
                        }
                    });
                }, {
                    threshold: 0.1,
                    rootMargin: '-20% 0px -70% 0px'
                });

                sections.forEach(section => observer.observe(section));
            }

            // Event listeners
            window.addEventListener('scroll', updateScrollProgress);
            document.addEventListener('DOMContentLoaded', () => {
                setupIntersectionObserver();
                updateScrollProgress();
            });

            // Handle clicks on navigation links
            document.querySelectorAll('a[href^="#"]').forEach(anchor => {
                anchor.addEventListener('click', function (e) {
                    e.preventDefault();
                    const targetId = this.getAttribute('href').substring(1);
                    scrollToSection(targetId);
                });
            });

            // Keyboard shortcuts
            document.addEventListener('keydown', function(e) {
                if (e.ctrlKey || e.metaKey) {
                    switch(e.key) {
                        case 'b':
                            e.preventDefault();
                            toggleNav();
                            break;
                    }
                }
            });

            // Auto-expand current section submenu on page load
            document.addEventListener('DOMContentLoaded', function() {
                // If there's a hash in URL, expand that section's submenu
                if (window.location.hash) {
                    const hash = window.location.hash.substring(1);
                    const variable = hash.split('-')[0];
                    if (variable && document.getElementById(`submenu-${variable}`)) {
                        toggleSubmenu(variable);
                    }
                }
            });
        </script>
    </body>
    </html>
    '''

    # Save HTML file
    with open(html_filename, 'w', encoding='utf-8') as f:
        f.write(html_content)

    print(f"✅ Simple HTML report created: {html_filename}")

    # ==========================================
    # CREATE ZIP FILES
    # ==========================================
    print(f"\n📦 Creating ZIP Archives...")

    zip_files = []
    zip_mapping = {
        national_dir: 'malaria_national_maps.zip',
        district_dir: 'malaria_district_maps.zip',
        subplot_dir: 'malaria_subplot_maps.zip'
    }

    for directory, zip_name in zip_mapping.items():
        zip_path = os.path.join(output_dir, zip_name)
        files = list(Path(directory).glob('*.png'))

        if files:
            with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
                for file_path in files:
                    zipf.write(file_path, file_path.name)

            zip_files.append(zip_path)
            print(f"   ✓ Created: {zip_name} ({len(files)} files)")

    # Create master ZIP with everything
    master_zip = os.path.join(output_dir, 'malaria_complete_analysis.zip')
    with zipfile.ZipFile(master_zip, 'w', zipfile.ZIP_DEFLATED) as zipf:
        for map_file in all_map_files:
            zipf.write(map_file, f"maps/{Path(map_file).name}")
        zipf.write(html_filename, 'malaria_trend_maps_report.html')

    print(f"   ✓ Created master archive: malaria_complete_analysis.zip")

    # FINAL SUMMARY
    print("\n" + "=" * 60)
    print("🎉 COMPREHENSIVE MALARIA TREND MAPPING COMPLETE!")
    print("=" * 60)

    print(f"🗺️  **Analysis Summary:**")
    print(f"   • Variables analyzed: {len(variables)} malaria indicators")
    print(f"   • Total maps generated: {len(all_map_files)}")
    print(f"   • Simple HTML report with embedded images")
    print(f"   • Geographic levels: National → District → Chiefdom")

    print(f"\n📊 **Variables Processed:**")
    for variable in variables:
        var_title = variable.replace('_', ' ').title()
        print(f"   • {var_title}")

    print(f"\n🗺️  **Maps Created:**")
    print(f"   • National overview maps: {len(variables)}")
    print(f"   • District-level maps: Multiple per variable")
    print(f"   • Chiefdom subplot maps: Multiple per district")
    print(f"   • Simple HTML report: 1 basic interface")

    print(f"\n📁 **All outputs saved in: {output_dir}/**")
    print("   Color scheme: Pink = Increasing incidence, Blue = Decreasing incidence")
    print("\n✨ **Ready for malaria trend analysis!**")

# EXECUTE THE FUNCTION
if __name__ == "__main__":
    print("🚀 MALARIA TREND MAPS GENERATOR - SIMPLE HTML FORMAT")
    print("=" * 60)
    print("Creating malaria trend mapping suite with basic HTML...")
    print()

    try:
        create_comprehensive_malaria_trend_maps()
    except Exception as e:
        print(f"\n❌ ERROR: {e}")
        print("\nPlease ensure:")
        print("• Required Python packages are installed: pandas, geopandas, matplotlib, numpy")
        print("• Input files exist at correct locations:")
        print("  - /content/2024_snt_data.xlsx")
        print("  - /content/Chiefdom2021.shp")
        print("• You have write permissions in the current directory")
