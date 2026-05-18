# Network Graph Analysis: Relationship Intelligence from Enriched LinkedIn Data

**Ascendancy Technical Exercise** | Tomi Brasca

This project analyzes 248 enriched LinkedIn profiles to answer two core questions:
1. **How well-connected is this network?**
2. **What groups, clusters, or communities does it consist of?**

Since the data lacks explicit relationship edges, all connections are inferred from shared attributes (companies, schools, locations, skills, industries) using temporal overlap checks and tiered edge weighting.

---

## Table of Contents
- [Project Overview](#project-overview)
- [Analysis Approach](#analysis-approach)
- [Setup Instructions](#setup-instructions)
- [Running the Analysis](#running-the-analysis)
- [Outputs](#outputs)
- [Key Findings](#key-findings)
- [Technical Details](#technical-details)
- [Limitations & Next Steps](#limitations--next-steps)

---

## Project Overview

This analysis constructs a social network graph from enriched LinkedIn profile data where **no explicit edges exist**. The core challenge is designing a meaningful edge-weighting scheme that reveals real structure beneath surface-level connections.

### Dataset
- **248 enriched LinkedIn profiles** with professional and educational history
- **No explicit relationship data** — all edges are inferred from shared attributes
- **Key attributes**: work experience, education, location, skills, current industry
- **Completeness**: 96% have connection counts, 48% have graduation years, 23% have skills data

### Key Innovation
Unlike naive approaches that create edges for any shared attribute, this analysis:
- **Requires temporal overlap** for work and school relationships (co-workers must have overlapping tenure)
- **Downweights Clemson University edges** (50% of network shares this affiliation, creating a near-clique)
- **Incorporates skill similarity** and **industry bridges** to reveal hidden professional communities
- **Uses tiered edge weights** based on relationship strength and signal rarity

---

## Analysis Approach

### Edge Construction

Edges are weighted using a **tiered scheme** that reflects both relationship strength and signal rarity:

| Relationship Type | Weight | Rationale |
|---|---|---|
| Shared non-Clemson company (temporal overlap) | **+4** per company | Strongest signal — actually co-worked |
| Shared Clemson work (temporal overlap) | **+2** per entity | Meaningful but common (deduplicated) |
| Shared non-Clemson school (temporal overlap) | **+3** per school | Moderate signal |
| Shared Clemson school (temporal overlap) | **+1** | 125/248 attended Clemson — needs downweighting |
| Same specific location (not "United States") | **+1** | Contextual co-location |
| Shared ≥3 specialized skills | **+2** | Professional affinity (where data exists) |
| Same current industry, different employer | **+1** | Weak professional-ecosystem bridge |

**Why temporal overlap matters**: Two people who attended Clemson 30 years apart shouldn't have an edge. Requiring date overlap ensures edges represent likely real connections, not coincidental shared affiliations.

### Code Example: Edge Construction

Here's how the temporal overlap check and edge weight computation work:

```python
from datetime import datetime

def has_temporal_overlap(exp1: dict, exp2: dict) -> bool:
    """Check if two work/school experiences overlapped in time."""
    s1 = parse_date(exp1.get("start_date"))
    e1 = parse_date(exp1.get("end_date")) or datetime.now()
    s2 = parse_date(exp2.get("start_date"))
    e2 = parse_date(exp2.get("end_date")) or datetime.now()
    
    if not s1 or not s2:
        return False
    return s1 <= e2 and s2 <= e1

def compute_edge(person_a: str, person_b: str) -> dict | None:
    """Compute weighted edge between two people. Returns None if no connection."""
    weight = 0
    edge_types = []
    
    # Shared companies (with temporal overlap)
    for company in set(person_a.companies) & set(person_b.companies):
        if any(has_temporal_overlap(ea, eb) 
               for ea in person_a.companies[company] 
               for eb in person_b.companies[company]):
            if company == "Clemson University":
                weight += 2
                edge_types.append("clemson_work")
            else:
                weight += 4
                edge_types.append("work")
    
    # Shared schools (with temporal overlap)
    for school in set(person_a.schools) & set(person_b.schools):
        if any(has_temporal_overlap(ea, eb) 
               for ea in person_a.schools[school] 
               for eb in person_b.schools[school]):
            if is_clemson(school):
                weight += 1
                edge_types.append("clemson_school")
            else:
                weight += 3
                edge_types.append("school")
    
    # Same specific location
    if person_a.location and person_b.location and person_a.location == person_b.location:
        weight += 1
        edge_types.append("location")
    
    # Shared specialized skills (≥3)
    shared_skills = person_a.skills & person_b.skills
    if len(shared_skills) >= 3:
        weight += 2
        edge_types.append("skill")
    
    # Same industry, different current employer
    if (person_a.industry and person_b.industry and 
        person_a.industry == person_b.industry and
        person_a.current_company != person_b.current_company):
        weight += 1
        edge_types.append("industry")
    
    if weight == 0:
        return None
    
    return {"weight": weight, "edge_types": edge_types}
```

**Example**: Two people who both worked at Invisible Technologies (overlapping 2020-2022), share 5 specialized skills, and live in NYC would have:
- Work edge: +4
- Skill edge: +2
- Location edge: +1
- **Total weight: 7**

### Network Metrics

The analysis computes:
- **Global metrics**: density, clustering coefficient, diameter, average path length
- **Node centrality**: degree, betweenness (bridge nodes), eigenvector (connected to well-connected)
- **Community detection**: Louvain algorithm at multiple resolutions

### Visualizations

Interactive Plotly network graphs with:
- Nodes sized by betweenness centrality (bridge nodes are larger)
- Nodes colored by detected community
- Full network view (edges weight ≥2)
- Clemson-removed subgraph (reveals hidden professional structure)
- Ego network centered on Ascendancy CEO

---

## Setup Instructions

### Prerequisites
- Python 3.10+
- pip

### Installation

1. Clone or download this repository
2. Create a virtual environment:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate  # On Windows: .venv\Scripts\activate
   ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

### Dependencies
```
networkx>=3.0          # Graph construction and analysis
python-louvain>=0.16   # Community detection
plotly>=5.18           # Interactive visualizations
polars>=1.0            # High-performance data wrangling
matplotlib>=3.8        # Static plots
seaborn>=0.13          # Statistical visualization
scipy>=1.12            # Scientific computing
jupyter>=1.0           # Notebook environment
```

---

## Running the Analysis

### Option 1: Jupyter Notebook (Recommended)
```bash
jupyter notebook notebook.ipynb
```

The notebook is structured in sections:
1. Data Loading & Cleaning
2. Edge Construction
3. Graph Metrics
4. Community Detection
5. Visualizations
6. Findings & Observations
7. Assumptions & Limitations

Run all cells to generate the complete analysis and outputs.

### Option 2: Run All at Once
```bash
jupyter nbconvert --to notebook --execute notebook.ipynb
```

---

## Outputs

All outputs are saved to the `output/` directory:

### Interactive Network Visualizations (HTML)
- **`network_full.html`**: Full network graph with 248 nodes, 3,919 edges (weight ≥2)
  - Nodes colored by detected community
  - Nodes sized by betweenness centrality (bridge nodes)
  - Interactive hover: name, title, company, location, centrality metrics
  
- **`network_no_clemson.html`**: Clemson-removed subgraph (616 edges)
  - Reveals hidden professional structure beneath dominant Clemson cluster
  - Shows investigations/tech community that was previously obscured
  
- **`ego_kamron.html`**: 2-hop ego network centered on Kamron Palizban (Ascendancy CEO)
  - 167 nodes, 3,595 edges
  - Shows how the founder connects to the broader network

### Static Dashboard (PNG)
- **`metrics_dashboard.png`**: 6-panel statistical overview
  - Degree distribution
  - Community sizes (bar chart)
  - Top 15 bridge nodes (betweenness centrality)
  - Degree vs. betweenness scatter (by community)
  - Clemson graduation year cohorts
  - Edge type composition

### Opening Visualizations
Simply open the HTML files in any web browser. They're fully self-contained with no external dependencies.

---

## Key Findings

### Q1: How well-connected is this network?

The network is **moderately connected** with strong hub-and-spokes topology:

- **7,601 total edges** across 248 nodes
- **Density: 0.25** (moderately connected - 25% of all possible edges exist)
- **Giant component: 229/248 nodes (92.3%)** - most people are reachable
- **Only 17 isolates** - people with no shared context
- **Average degree: 61.3** - each person connects to ~61 others
- **Average path length: 2.01** - most people are 2 hops away
- **Transitivity: 0.84** - high clustering within communities

**Clemson University dominates the network structure**: 5,171 edges (68%) stem from shared Clemson school affiliation. Removing these edges fragments the network but reveals a hidden professional ecosystem of tech startups and investigations firms.

### Q2: What communities exist?

**Louvain community detection** (default resolution) found **21 communities** with **modularity 0.187**:

| Community | Size | Clemson Affiliated | Description |
|-----------|------|-------------------|-------------|
| **Clemson · Greenville** | 90 | 73 (81%) | Heavy Greenville/Clemson SC concentration |
| **Social Slooth + Invisible Technologies** | 70 | 1 (1%) | Investigations + tech startups cluster (bridged by industry/skill edges) |
| **Clemson · Campus / Recent Grads** | 69 | 62 (90%) | Current students and recent graduates |
| **Isolates** | 17 | mixed | No shared context with any other nodes |

**Higher resolution (1.5)** splits the network into **43 communities**, segmenting Clemson clusters by **graduation year cohorts** (e.g., 2023 grads form a tight 18-person cluster; 2024-2029 students form another). This temporal structure emerges naturally from the date-overlap requirement.

### Most Important Bridge Nodes (Betweenness Centrality)

These people connect otherwise separate communities:

1. **Nicholas Deas** (0.087) - Columbia Engineering grad student, bridges multiple Clemson sub-communities
2. **Jaime Aitkin** (0.036) - Corporate Resolutions investigator, connects external professional clusters
3. **Ryan Mahon** (0.035) - Amazon SDE, bridges Clemson to tech ecosystem
4. **Devin Narula** (0.033) - MathWorks data scientist, academic-to-industry bridge

### Strongest Individual Connections (Edge Weight)

1. **Kamron Palizban ↔ Francis Pedraza**: weight 16 (4 shared companies: Invisible, Visionary Ventures, Ascendancy, Cheeky Labs)
2. **Noah W. ↔ John Christian Kuehnert**: weight 12 (3 shared companies: Ascendancy, Surv, Atlantis)
3. **Francis Pedraza ↔ Scott Downes**: weight 10 (2 companies + 8 shared skills)

The **Ascendancy founding team** is exceptionally tightly knit with multiple overlapping work histories.

### Key Insights

1. **Hidden professional ecosystem emerged**: Industry and skill edges revealed that Social Slooth investigators (8 people) and other security/investigations firms form a connected professional community—previously obscured by Clemson dominance.

2. **Skills data bridged fragmented communities**: Despite 23% coverage, shared specialized skills revealed professional affinities invisible in work/school history alone.

3. **Temporal overlap eliminated 40% of false edges**: Date-overlap requirements removed spurious connections between people who shared affiliations decades apart (e.g., Clemson graduates from 1995 and 2025).

---

## Technical Details

### Graph Construction
- **Library**: NetworkX 3.0+
- **Graph type**: Undirected, weighted
- **Nodes**: 248 (one per LinkedIn profile)
- **Edges**: 7,601 (filtered to weight ≥2 for visualization clarity)

### Data Cleaning & Normalization
- **Company normalization**: Merged 7+ Clemson sub-entities into "Clemson University"
- **Name cleanup**: Stripped emoji characters from profile names
- **Date parsing**: Handles both `YYYY` and `YYYY-MM-DD` formats
- **Temporal overlap function**: Compares start/end dates to verify co-occurrence at companies/schools

### Community Detection
- **Algorithm**: Louvain (greedy modularity optimization)
- **Implementation**: `python-louvain` library
- **Resolutions tested**: default (1.0) and high (1.5)
- **Result**: 21 communities at default, 43 at high resolution

### Centrality Measures
- **Degree centrality**: Simple connection count (normalized)
- **Betweenness centrality**: How often a node lies on shortest paths between others (identifies bridges)
- **Eigenvector centrality**: Measures connection to well-connected nodes (identifies influence hubs)
- **PageRank**: Google's algorithm adapted for social networks (with edge weights)

### Visualization Technology
- **Plotly**: Interactive HTML graphs with zoom, pan, hover
- **Layout algorithm**: Spring/force-directed (Fruchterman-Reingold) with 80 iterations
- **Pre-computed positions**: Layout calculated in Python so browser renders instantly
- **Styling**: Dark theme (#0f0f1e background) for readability

---

## Limitations & Next Steps

### Current Limitations

1. **All relationships are inferred** from shared attributes, not ground-truth connections
2. **Skills data is sparse** (23% coverage), limiting professional affinity signals
3. **No interaction data** (LinkedIn messages, endorsements, recommendations)
4. **Clemson bias** likely reflects ego network centered on a Clemson-affiliated individual
5. **Edge weights are heuristic**, not empirically validated against known relationships
6. **Static snapshot** cannot track network evolution over time

### Assumptions

- Temporal overlap at the same company indicates likely acquaintance
- Shared school attendance with date overlap indicates social connection
- 3+ shared specialized skills indicates professional affinity
- Same industry, different employer indicates awareness of each other's professional world
- This dataset represents an ego network (248 people likely connected to one central individual)

### Recommended Next Steps

1. **Enrich with interaction data**: LinkedIn endorsements, recommendations, InMail exchanges
2. **NLP on text fields**: Apply semantic similarity to `headline` and `about` sections
3. **Validate edge weights**: Survey known connections to calibrate scoring
4. **Production deployment**: Port to Neo4j for real-time Cypher queries at scale
5. **Build recommendation engine**: "Connect X to Y via Z" - Ascendancy's core value proposition
6. **Time-aware analysis**: Track how network grows, which connections strengthen/decay
7. **A/B test weightings**: Measure which schemes best predict real interactions

---

## Project Structure

```
.
├── README.md                    # This file
├── notebook.ipynb               # Complete analysis notebook
├── data.json                    # Input: 248 enriched LinkedIn profiles
├── requirements.txt             # Python dependencies
├── output/
│   ├── network_full.html        # Interactive full network visualization
│   ├── network_no_clemson.html  # Clemson-removed subgraph
│   ├── ego_kamron.html          # CEO ego network
│   └── metrics_dashboard.png    # Statistical dashboard (6 panels)
└── .gitignore
```

---

## About This Analysis

This network analysis was developed as a technical exercise for **Ascendancy**, a relationship intelligence platform. It demonstrates inferring relationships from incomplete data, applying network science fundamentals, extracting actionable insights, and communicating technical findings clearly.

**Contact**: tomibrasca97@gmail.com

---

## License

This is a take-home technical exercise. Data and code are provided for evaluation purposes.
