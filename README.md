# BayeSMART: Bayesian clustering of multi-sample spatially resolved transcriptomics data

The model consists of two part: 1) Spatial domain identification of spatial transcriptomics(ST) on multi-sample and 2) bulk RNA-seq deconvolution

![BayeSMART](fig/flowchart.png)

## Spatial Domain Identification for Multi-sample
iBURST was developed and tested under R 4.2.2.

The code for the first part is in folder code/mIMPACT. Clone the whold repo and run codes in

``` 
main.R
```

For getting the results of six breast cancer patients, source the functions first by
```r
source("/Users/yanghongguo/Desktop/code/BayeSMART.R")
```

import the data and preprocess to get the molecular, image, and spatial profiles
```r
load("/Users/yanghongguo/Desktop/Research/iBurst/data/4 dataset final/HER2_p.RData")

xys <- lapply(xys_her2, function(info.i){
  as.matrix(info.i[, c("x", "y")])
})

# molecular profile
Y <- batch_remove(cnts_her2, xys=xys,  gene_select = "hvgs", n_gene = 2000, pcn = 3)

# image profile
V <- matrix(ncol = 3, nrow = 0)
for (s in 1:4){
  # filename = paste0(path, my_dict[[s]], ".RData")
  # load(filename)
  
  cell_info <- cell_info_her2[[s]]
  
  cell_xy <- cell_info[, 1:2]
  cell_name <- cell_info$name
  
  V1 <- get.cell.abundance(cell_xy, cell_name, xys[[s]], lattice = 'square')
  V1 <- new.cell.abundance(V1) # group cell types to only three kinds: tumor, stroma, and immune
  
  V <- rbind(V, V1)
}

# spatial profile
spatial_info <- spatial_concat(xys, n_neighbor = 8)
G <- spatial_info$G
G_origin <- spatial_info$G_origin
```

Then run the model
```r
# number of spatial domains
K <- 6

# weight of image profile
w <- 1/10

# run the miIMPACT model for spatial domain identification on multi-sample
result <- run.BayeSMART(V, Y, G, n_cluster = K, w = w)

# get the posterior inference on spatial domains
spatial_domain <- get.spatial.domain(result)
z_all <- domain_split(spatial_domain, G_origin, xys)
```


Calculate the ARI for each sample
```r
library(mclust)
scores_ARI <- c()

for (s in 1:4){
  spatial_domain_refined <- z_all[[s]]
  real_region <- xys_her2[[s]]$annotation
  
  ari_score <- adjustedRandIndex(spatial_domain_refined, real_region)
  scores_ARI <- c(scores_ARI, ari_score)
}

print(scores_ARI)
```

The result is
```r
> scores_ARI
[1] 0.6744486 0.3044600 0.7814962 0.3578727
```
