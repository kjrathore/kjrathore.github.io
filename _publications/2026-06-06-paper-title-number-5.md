---
title: "Hybrid Modeling of Harmful Algal Blooms in the Gulf of Maine Using Ocean Color and Regional Ocean Models"
collection: publications
permalink: /publication/2026-06-roms
excerpt: 
date: 2026-06-06
venue: 'Earth Arxiv'
paperurl: 'https://sciety.org/articles/activity/10.31223/x5br2v'
citation: 'Rathore, K. J., Buckner, J. H., Watson, J. R. (2026). Hybrid Modeling of Harmful Algal Blooms in the Gulf of Maine Using Ocean Color and Regional Ocean Models'
---

Harmful algal blooms (HABs) in coastal waters contaminate shellfish, close fisheries, and threaten public health, yet the physical and biological conditions that trigger a bloom are rarely captured at sufficient resolution by any single observing system. This paper develops a hybrid machine learning framework for species-level HAB detection in the Gulf of Maine, combining satellite ocean color, in-situ flow imaging, and regional ocean forecast model outputs.

**Key Innovation**: Rather than relying on satellite observations alone, we take a simulation-as-features approach: physical oceanographic variables such as sea surface temperature and salinity, drawn from a regional ocean forecast model, are used to enrich a classifier trained on satellite ocean color signal. This lets the model draw on mechanistic oceanographic context even where remote sensing alone is ambiguous or gap-filled.

**Demonstration**: We build a machine learning classifier for five HAB-forming phytoplankton species in the Gulf of Maine, using three input sources: satellite ocean color observations, continuous in-situ cell-count records from automated flow imaging instruments at a fixed coastal site, and physical oceanographic features from a regional ocean forecast model.

**Results**: Enriching the satellite signal with forecast model features consistently improved species-level bloom detection. The best hybrid configuration achieved a median F1 score of 0.745 across taxa. We also found that ocean color quality and physical oceanographic context act as partially substitutable performance levers, meaning gains from one input source can offset losses from the other.

**Impact**: This approach gives coastal monitoring programs a practical path to more reliable species-level HAB detection, particularly in settings where satellite data quality is inconsistent or in-situ sampling is sparse. It also clarifies which input sources can compensate for gaps in others, informing deployment decisions for coastal observing infrastructure.

**Authors**: Kunal J. Rathore, John H. Buckner, James R. Watson  
**Affiliation**: Oregon State University, College of Earth, Ocean, and Atmospheric Sciences