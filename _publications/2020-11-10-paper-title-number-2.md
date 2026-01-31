---
title: "RA2Vec: Distributed Representation of Protein Sequences with Reduced Alphabet Embeddings"
collection: publications
permalink: /publication/2020-09-21-ra2vec
excerpt: 
date: 2020-09-21
venue: 'Proceedings of the 11th ACM International Conference on Bioinformatics, Computational Biology and Health Informatics (ACM-BCB 2020)'
paperurl: 'https://dl.acm.org/doi/10.1145/3388440.3414925'
citation: 'Wijesekara, R. Y., Lahorkar, A., Rathore, K., & Valadi, J. K. (2020). RA2Vec: Distributed representation of protein sequences with reduced alphabet embeddings. In Proceedings of the 11th ACM International Conference on Bioinformatics, Computational Biology and Health Informatics (pp. 1-1).'
---

Protein function identification has become an important task due to the rapid growth of sequenced genomes. This work introduces RA2Vec (Reduced Alphabets to Vectors), a novel approach for protein sequence representation that combines reduced amino acid alphabets with Word2Vec-style embeddings. We map protein sequences to reduced alphabets based on hydropathy and conformational similarity, then apply skip-gram models to create distributed vector representations that capture both biochemical properties and sequential context.

**Key Innovation**: Rather than using the full 20-amino-acid alphabet, RA2Vec reduces dimensionality through biochemically meaningful alphabet reduction schemes. These reduced representations are embedded using skip-gram models, creating vectors that encode semantic/syntactic information similar to NLP word embeddings.

**Results**: Experimental validation across protein function prediction tasks using Support Vector Machines demonstrated that reduced alphabet representations can outperform full-alphabet approaches in certain classification tasks. Synergistic combinations of different embedding types (original ProtVec + hydropathy + conformational) yielded the best performance, showing that biochemically informed dimensionality reduction improves predictive power while reducing computational complexity.

**Impact**: This work demonstrated that NLP techniques transfer successfully to computational biology when adapted to domain-specific constraints, laying groundwork for more efficient protein function annotation pipelines.

**Authors**: Rajitha Yasas Wijesekara, Ashwin Lahorkar, Kunal Rathore, Jayaraman Krishnamoorthy Valadi  
**Affiliation**: Centre for Modeling & Simulation, Savitribai Phule Pune University, India
