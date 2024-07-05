# iML1515_flux_balance_analysis
iML1515 - E.coli model from BIGG MODELS Database, using this to perform constraint-based modelling and Flux Balance to optimize the model and visualize the results.
Link to E.coli model to download - http://bigg.ucsd.edu/models/iML1515

#Dependencies and packages to install
# Step 1: Set up the environment
pip install cobra
pip install openpyxl  # For saving results to Excel
pip install python-docx  # For creating Word documents
pip install matplotlib seaborn  # For visualizations
pip install xlsxwriter  # For supporting MultiIndex in Excel

# Step 2 Load your model and print it's summary
import cobra
from cobra.io import load_matlab_model
model = load_matlab_model("/path_to_your_model/iML1515.mat")

# Print the model summary to verify it has loaded correctly
print(model.summary())

## YOU CAN CHANGE THE OBJECTIVE REACTION TO STUDY YOUR REACTION OF INTEREST ##

# Objective Coefficient - # Update the objective coefficient for biomass reaction
model.objective = 'BIOMASS_Ec_iML1515_core_75p37M'
model.reactions.get_by_id('BIOMASS_Ec_iML1515_core_75p37M').objective_coefficient = 1.0

## THESE ARE THE SIGNIFICANT REACTIONS AND KEY PATHWAYS, YOU CAN REDUCE/INCREASE THEM TO STUDY A WIDER OR SPECIFIC PATHWAY/ REACTION ##

# Significant reactions
# Define significant reactions and key pathways for E. coli iML1515 model
significant_reactions = [
    'BIOMASS_Ec_iML1515_core_75p37M', 'ATPM', 'PGI', 'PFK', 'PYK', 'TPI', 'GAPD', 'PGK', 'PGM', 'ENO', 'PYR', 'LDH_D',
    'G6PDH2r', 'GND', 'PGL', 'RPI', 'RPE', 'TKT1', 'TKT2', 'TALA'
]

# Key Pathways
key_pathways = {
    'glycolysis': ['PGI', 'PFK', 'TPI', 'GAPD', 'PGK', 'PGM', 'ENO', 'PYK'],
    'TCA_cycle': ['ACONTa', 'ACONTb', 'AKGDH', 'CS', 'ICDHyr', 'SUCDi', 'SUCOAS']
}


### FOR THE BELOW CONDITIONS, YOU CAN CHANGE PARAMETERS ACCORDING YOU YOUR NEEDS TO SIMULATE THE ENVIRONMENT ###

# Set up a range of conditions for E. coli model
conditions = {
    'Aerobic High O2': {'EX_o2_e': (-20, 1000)},
    'Aerobic Medium O2': {'EX_o2_e': (-10, 1000)},
    'Aerobic Low O2': {'EX_o2_e': (-5, 1000)},
    'Anaerobic': {'EX_o2_e': (0, 0)},
    'High Glucose': {'EX_glc__D_e': (-20, 1000)},
    'Medium Glucose': {'EX_glc__D_e': (-10, 1000)},
    'Low Glucose': {'EX_glc__D_e': (-5, 1000)},
    'No Glucose': {'EX_glc__D_e': (0, 0)},
    'High Glycerol': {'EX_glyc_e': (-20, 1000)},
    'Medium Glycerol': {'EX_glyc_e': (-10, 1000)},
    'Low Glycerol': {'EX_glyc_e': (-5, 1000)},
    'No Glycerol': {'EX_glyc_e': (0, 0)},
    'High Acetate': {'EX_ac_e': (-20, 1000)},
    'Medium Acetate': {'EX_ac_e': (-10, 1000)},
    'Low Acetate': {'EX_ac_e': (-5, 1000)},
    'No Acetate': {'EX_ac_e': (0, 0)},
    'High Ammonium': {'EX_nh4_e': (-20, 1000)},
    'Medium Ammonium': {'EX_nh4_e': (-10, 1000)},
    'Low Ammonium': {'EX_nh4_e': (-5, 1000)},
    'No Ammonium': {'EX_nh4_e': (0, 0)}
}

### FOR THE ABOVE CONDITIONS, YOU CAN CHANGE PARAMETERS ACCORDING YOU YOUR NEEDS TO SIMULATE THE ENVIRONMENT ###


# Perform FBA under different conditions and store results
results = {}

for condition, bounds in conditions.items():
    with model:
        # Set the objective function to E. coli biomass reaction
        model.objective = 'BIOMASS_Ec_iJO1366_WT_53p95M'

        for reaction_id, (lower_bound, upper_bound) in bounds.items():
            model.reactions.get_by_id(reaction_id).bounds = (lower_bound, upper_bound)

        solution = model.optimize()

        # Collect results and additional debug information
        if solution.status == 'optimal':
            biomass_flux = solution.fluxes.get('BIOMASS_Ec_iJO1366_WT_53p95M', 0.0)
            growth_rate = solution.objective_value

            results[condition] = {
                'growth_rate': growth_rate,
                'biomass_flux': biomass_flux,
                'reaction_fluxes': {rxn: solution.fluxes.get(rxn, 0.0) for rxn in significant_reactions},
                'pathway_fluxes': {pathway: {rxn: solution.fluxes.get(rxn, 0.0) for rxn in rxns} for pathway, rxns in key_pathways.items()}
            }
        else:
            warnings.warn(f"Solver status is '{solution.status}'. Condition '{condition}' is infeasible.", UserWarning)

# Print and save results for debugging
output_file = "ecoli_iJO1366_fba_results.txt"
with open(output_file, 'w') as f:
    for condition, result in results.items():
        f.write(f"Condition: {condition}\n")
        f.write(f"Growth rate: {result['growth_rate']}\n")
        f.write(f"Biomass flux: {result['biomass_flux']}\n")
        for rxn, flux in result['reaction_fluxes'].items():
            f.write(f"{rxn}: {flux}\n")
        f.write("\n")
print(f"Debug results saved to '{output_file}'")


# Visualize all reaction fluxes for feasible conditions and save plots
for condition, result in results.items():
    flux_data = []
    for reaction, flux in result['reaction_fluxes'].items():
        flux_data.append({
            'Condition': condition,
            'Reaction': reaction,
            'Flux': flux
        })
    flux_df = pd.DataFrame(flux_data)

    plt.figure(figsize=(12, 6))
    sns.barplot(data=flux_df, x='Condition', y='Flux', hue='Reaction', palette='viridis', dodge=False, legend=False)
    plt.title(f'Reaction Fluxes under {condition}')
    plt.xticks(rotation=90)
    plt.tight_layout()
    plt.savefig(f'reaction_fluxes_{condition}_iJO1366.png')
    plt.close()


# Generate a detailed report
doc = Document()
doc.add_heading('Metabolic Pathway Analysis Report (E. coli - iJO1366)', level=1)

for condition, result in results.items():
    doc.add_heading(condition, level=2)
    doc.add_paragraph(f'Growth rate: {result["growth_rate"]}\n')
    doc.add_paragraph(f'Biomass flux: {result["biomass_flux"]}\n')

    doc.add_heading('Significant Reaction Fluxes', level=3)
    for rxn, flux in result['reaction_fluxes'].items():
        doc.add_paragraph(f'{rxn}: {flux}')

    doc.add_heading('Key Pathway Fluxes', level=3)
    for pathway, fluxes in result['pathway_fluxes'].items():
        doc.add_heading(pathway, level=4)
        for rxn, flux in fluxes.items():
            doc.add_paragraph(f'{rxn}: {flux}')

    # Add plots for each condition
    doc.add_heading('Visualizations', level=3)
    doc.add_picture(f'reaction_fluxes_{condition}_iJO1366.png', width=Inches(6.0))

# Save the report to a file
report_path = 'ecoli_metabolic_pathway_analysis_report_iJO1366.docx'
doc.save(report_path)
print(f"Metabolic pathway analysis report for E. coli iJO1366 saved to '{report_path}'")

