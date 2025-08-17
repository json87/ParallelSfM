# ParallelSfM

This project implements a parallel SfM solution for sparse reconstruction of large-scale UAV images. The core ideas are a local connection-constrained edge weighting strategy for match graph construction and an anchor-free parallel merging algorithm for the merged model generation. The solution has been verified by using a dataset containing ninety thousand images over an area of 50.0 $km^2$.

![sfm-8w](https://github.com/json87/ParallelSfM/blob/main/doc/sfm-8w.png)

## NOTICE!

For the current version, cameras should be calibrated and their parameters are fixed during BA since the merge step does not optimize them.

## Build

The executable file has been provided in the bin directory.

## Dataset

The test dataset has been provided via the [google driver](https://drive.google.com/drive/folders/1t0IXWA5YQXxKyUYmjDELwSOYCB6UnLD3?usp=drive_link) and [Baidu disk](https://pan.baidu.com/s/1vUYfQQnagc2S00ZXPai8GQ?pwd=pk9p), and the used configuration file is in the test directory of this repo.

## Usage

The scripts to run the whole process of the ParallelSfM are given:

### 1. Project generation
```sh
jsonsfmx project_generator  --prj_name $project_name --prj_location $project_location --group_file $r3m_file 
```
- ```$project_name```:   The name of the project
- ```$project_location```: The directory where the project is stored
- ```$r3m_file ```:  The file whose suffix is r3m, which mainly sets the image storage directory and image camera parameters in the project. Before executing the command shown, please set up the r3m file.

### 2. Feature extraction
```sh
jsonsfmx feature_extractor --prj_file $prj_r3m_file --SiftExtraction.use_gpu $use_gpu --SiftExtraction.max_num_features $max_num_features --SiftExtraction.max_image_size $max_image_size
```
- ```$prj_r3m_file```:   The new r3m file generated in the previous step, which is stored in the prj_location set in the previous step.

- ```$use_gpu```:  Whether to use the GPU. The default value is 1.

- ```$max_num_features ```:  Maximum number of features to detect, keeping larger-scale features. The default value is 8192.

- ```$max_image_size ```:  Maximum image size, otherwise image will be down-scaled.. The default value is 3200.

### 3. Codebook generation

If you already have a codebook, skip this step. The function of codebooks is to assist in image retrieval and feature matching. Codebooks can be generated from the dataset being applied or from other datasets.

```sh
jsonsfmx vlad_index_builder --database_path $database_path --vocab_tree_path $output_path --num_visual_words $num_visual_words --max_num_images $max_num_images --max_num_features $max_num_features
```
- ```$database_path```: Path of the db file generated in Step 1.
- ```$output_path```: Output path of codebook file. Note that the path must be specified to the file name rather than the directory where the file is located. You can set no suffix for the file.
- ```$num_visual_words```: The number of words in the codebook. The larger the value, the higher the retrieval precision and the lower the retrieval efficiency. The default value is 256.
- ```$max_num_images````: The maximum number of images used to train the codebook. The default value is -1. That is all images are used.
- ```$max_num_features````: The maximum number of features for each image used to train the codebook. The default value is -1. That is all images are used.

### 4. Image retrieval
```sh
jsonsfmx vlad_index_retriever --database_path $database_path --vocab_tree_path $codebook_path --output_result_path $output_path --num_images $num_images
```
- ```$database_path```: Path of the db file generated in Step 1.

- ```$codebook_path```:  Path of the codebook file.

- ```$output_path```:  Path of the retrieval result. Note that the path must be specified to the file name rather than the directory where the file is located. You can set no suffix for the file.
- ```$num_images```:  The number of retrieval results for each query images. The default value is -1. That is all images are retireved.

### 5. Feature matching
```sh
jsonsfmx image_pair_matcher --database_path $database_path --ImagePairsMatching.match_list_path $match_list_path 
```
- ```$database_path```: Path of the db file generated in Step 1.

- ```$match_list_path```:  The path of the retrieval results obtained in the previous step.

### 6. Parallel reconstruction
```sh
jsonsfmx distributed_mapper
--output_path $output_path 
--database_path $database_path 
--image_path $images_path 
--Mapper.ba_refine_focal_length $refine_fl 
--Mapper.ba_refine_principal_point $refine_pp 
--Mapper.ba_refine_extra_params $refine_ep 
--ba_final_enabled $final_BA 
--dump_intermediate $dump_intermediate 
--image_overlap $image_overlap 
--max_num_images $max_num_images 
--weight_strategy $weight_strategy
--cluster_weight_ratio $cluster_weight_ratio 
--merge_strategy $merge_strategy 
--max_common_3d_points $max_common_3d_points
```
- ```$output_path```: output directory of reconstruction information and results
- ```$database_path```:  Path of the db file generated in Step 1.
- ```$images_path```:  Image directory.
- ```$refine_fl```:  Whether to optimize focal length during the reconstruction. The default value is 1.
- ```$refine_pp```:  Whether to optimize principal point during the reconstruction. The default value is 0.
- ```$refine_ep```:  Whether to optimize extra params during the reconstruction. The default value is 1.
- ```$final_BA```:  Whether or not conduct final bundle adjustment after merge. The default value is 0.
- ```$dump_intermediate```:  Whether or not dump intermediate reconstructions. The default value is 0.
- ```$image_overlap```:  The number of overlapping images between two clusters. The default value is 50.
- ```$max_num_images```:  The maximum number of images in a cluster. The default value is 500.
- ```$weight_strategy```:  Strategies for image clustering. There are three strategies, inlier_number, inlier_overlap, inlier_spatial. Inlier_number only considers the number of matching points between all images. inlier_overlap takes into account the number of matching points between images and the overlap area. inlier_spatial only considers the number of matching points between local images. The default value is inlier_overlap.
- ```$cluster_weight_ratio```:  If weight_strategy is inlier_number or inlier_spatial, set it to 0. if inlier_overlap, set it to 0.5.
- ```$merge_strategy```:  Strategies for cluster merging. There are seven strategies, exhaus_untriated, exhaus_edgeweight, exhaus_parallel, global_untriated, global_3dpoint, global_alignerror, global_alltracks. The default value is exhaus_edgeweight.
- ```$max_common_3d_points```:  The maximum number of common 3D points for cluster merging. Initial common 3D points are selected via reprojection errors. The default value of -1 means that no limit is set.


## Change log

- 2023.12.8
  - A new clustering method, inlier_spatial, has been added. This method currently has better clustering performance.
  - A new merging method, exhaus_parallel, has been added. This method has better performance for large datasets.

## Reference

```
@article{jiang2024efficient,
  title={Efficient structure from motion for UAV images via anchor-free parallel merging},
  author={Jiang, San and Ma, Yichen and Jiang, Wanshou and Li, Qingquan},
  journal={ISPRS Journal of Photogrammetry and Remote Sensing},
  volume={211},
  pages={156--170},
  year={2024},
  publisher={Elsevier}
}

@article{jiang2022parallel,
  author={Jiang, San and Li, Qingquan and Jiang, Wanshou and Chen, Wu},
  title={Parallel Structure From Motion for UAV Images via Weighted Connected Dominating Set}, 
  journal={IEEE Transactions on Geoscience and Remote Sensing}, 
  volume={60},
  pages={1-13},
  year={2022},
  publisher={IEEE}
}
```
