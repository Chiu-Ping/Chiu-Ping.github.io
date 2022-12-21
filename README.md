# Qiuping Shen -- CV

# Full Optimization Procedures

- Check the feature variables

  - Plot the distributions of those kinematics for those samples used in the multilabel training

- Generate those samples by kl-reweight method

  - Create the reweighted samples and make a copy to the proper directory

  ```bash
  cd run &&
  python3 ../scripts/derive_klreweight.py store_npy_wosm kl0p0 ggf ../reweightedsample
  ln -s ../../hhbbyy-optimization/reweightedsample/preselectec_ggfkl0p0_mc16a.npy ./
  ```

- Train the multi classification model:

  - training: 

  ```bash
  python3 bbyyoptimization.py -c json/multiclassification/base_multiclass.json -o check_json --train
  ```

  - validation: 

  ```bash
  python3 bbyyoptimization.py -c json/multiclassification/base_multiclass.json -o check_json --train -m check_json/booster_multiclass_20220831.dat --validate
  ```

  - bdtscore and corralation: 

  ```bash
  python3 bbyyoptimization.py -c json/multiclassification/base_multiclass.json -o check_json --train --bdtscore --corrmat
  ```

- Apply the multi classifier to all samples: the output named ```optimized_*_mc16[a,d,e],npy```

  ```bash
  python3 bbyyoptimization.py -c json/multiclassification/base_multiclass.json -o check_json -m check_json/booster_multiclass_20220831.dat --apply -a check_json/store_xgb
  ```

- Train BDT models for five categories with five HTCondor jobs:

  ```bash
  cd utils && 
  python3 submit_sbdt.py -o sbdt_data -j json/binaryclassification/devefive --train
  ```

- Apply five models to each sample and create the ```bdt_score``` variables: the output named ```xgboost_*_mc16[a,d,e],npy```

  ```bash
  python3 binaryoptimization.py -c json/binaryclassification/devefive/bin_zero.json -o sbdt_data -p check_json/store_xgb
  ```

- Plot the BDT score distribution and derive the best cut acoording to the significance and events at data sideband: the bdt cut value will be saved in a ```json``` file:

  ```bash
  python3 ../scripts/plot_sbdt_score.py sbdt_data/store_npy_bdt sbdt_data 
  ```

- Derive the table of Events and Yields as well as Cut Efficiency: two parameters( output folder and input path)

  ```bash
  python3 ../scripts/derive_table_cut.py sbdt_data/store_npy_bdt sbdt_data
  ```

- Apply the best BDT cut with BDT score samples to create the ```xgboost_fiveCat``` variables as labels:  the output named ```bdt_*_mc16[a,d,e].npy```

  ```bash
  python3 binaryoptimization.py -c json/binaryclassification/devefive/bin_zero.json -o sbdt_data --applybdt -t sbdt_data
  ```

- Convert the above numpy sample to root files for making the histogram used to fit to get modeling

  ```bash
  python3 ../scripts/npy_to_root.py sbdt_data/store_bdt_best sbdt_data/store_bdt_root
  ```

# The above command for categorization optimization



In the final, one could go to next step for fitting signal and backround modeling, also yields for five categories. After that, making the config file and then creating the workspace, performing the statistical fit.

- Collect the yields for each sample on the directory:```hhyybb-mxaodprocessor```:

  - Get yields for all samples:

    ```bash
    mkdir yields_opt && 
    python optimization/GetYields.py -i ../hhyybb-optimization/run/sbdt_data/store_bdt_best -o yields_opt
    ```

  - Fit the yields for all kl points:

    ```bash
    python optimization/FitYieldsForKL.py -i yields_opt
    ```

  - Get the VBF yields with linear combination method:

    ```bash
    python optimization/LinearComYields.py -i ../hhyybb-optimization/run/sbdt_data/store_bdt_best -o yields_opt
    ```


- Make the histogram with ```code/AnalysisFramework```: one need to pay attention to the selection inside ```json/H027/Optimization``` 

  - NOTE: NEED to move the selection to the part before applying the model, othersise, finally less events remained to do the fit.

  ```bash
  ./hhbbyy json/H027/Optimization/DataRun2.js
  ./hhbbyy json/H027/Optimization/DiHiggs.js
  ./hhbbyy json/H027/Optimization/SingleHiggs.js
  ./hhbbyy json/H027/Optimization/GammaGamma.js
  python MergeHHandH.py
  ```

- Derive the signal and background modeling ```hhyybb-background``` and ```hhyybb-signal```:

  - Directly fit the data sideband myy distribution with exponential function.
  - Fit the myy distribution of Single Higgs and DiHiggs with DCSB function.

  ```bash
  source setup.sh && 
  python BackgroundExponentialFit.py
  source run.sh
  ```

- Create the config file with XML format under ```hhyybb-config```: the config save in ```config``` folder.

  - Choose a certain version: ```four Categories``` or ```five Categories```

  ```bash
  source setup.sh && 
  python config_maker_fiveCat.py -i v.05 -o new_klcvv
  ```

- Generate the workspace under the directory ```xmlAnaWSBuilder```: the workspace saved in ```workspace``` foldes.

  - Softlink with config file.

  ```bash
  cd config && 
  ln -s ../../hhyybb-configmaker/config/new_klcvv ./
  ./exe/XMLReader -x config/new_klcvv/Steering_new_klcvv.xml
  ```

- Perform the statistic fit with the workspace under ```quickFit```:

  - To create condor job ```Parameter.condor``` including all points of kl and c2v.
  - To submit job with HTCondor containing multi-jobs.
  - To generate the limits with text or csv ```ggf_vbf_kl_cvv.txt``` at a certain format.

  ```bash
  python KLC2V_Limit_Scan.py --condor
  condor_submit Parameter.condor
  python KLC2V_Limit_Scan.py -n outname.csv
  ```

- Create the contour plots with the above outputs under ```hhyybb-contour```:

  - Make softlink for the limits txt/csv files
  - Only get the 2D contour limits with kl and c2v
  - Comparison with two limits results

  ```bash
  cd data && 
  ln -s ../../quickFit/hhyybb_scripts/limits_KL_CVV/ggf_vbf_kl_cvv.txt ggf_vbf_kl_cvv.txt
  source setup.sh && 
  python interpretation.py -i data/ggf_vbf_kl_cvv.txt
  python interpretation.py -i data/20220312.kl_c2v.csv,data/ggf_vbf_kl_cvv.txt -l baseline,optimization- 
  ```

# quickLimit to fit official workspace

- For VBF HH signal strength:

```bash
quickLimit -f /lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/workspace/WS-bbyy-non-resonant_param.root -o limit_asimov_mu0_0p0.txt -p mu_XS_HH_VBF -d asimovData_1
```

- For ggF HH signal strength:

```bash
quickLimit -f /lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/workspace/WS-bbyy-non-resonant_param.root -o limit_asimov_mu0_0p0.txt -p mu_XS_HH -d asimovData_1
```

- Complete the slides in three hours

  Baiygan

- Make the inputs for XML maker

  - numpy => Tree format => variable myy mjj weight for signal modeling
  - numpy => Text format => variable myy data sideband for background modeling
  - Run signal and background modeling

```bash
runSkimAndSlim_yybb ../source/HGamCore/HGamTools/data/SkimAndSlim_yybb.cfg /eos/atlas/atlascerngroupdisk/phys-higgs/HSG1/MxAOD/h026/mc16a/Nominal/mc16a.PowhegPy8_ttH125_fixweight.MxAODDetailed.e7488_s3126_r9364_p4180_h026.root/mc16a.PowhegPy8_ttH125_fixweight.MxAODDetailed.e7488_s3126_r9364_p4180_h026.001.root SampleName: mc16a.PowhegPy8_ttH125_fixweight OutputDir: ./ttH
```

# Updated Procedure with Preslected Sample included

- Yields for the reweighted sample kl=5.6 have some differences between Alkaid and Mine

  - Alkaid: df = pd.read_csv('preselected_13_11_2022/arrays/hh_bbyy_ggF_kl_5p6.csv') ==> <font color=red>***5.1855904940280935***</font>
  - Mine: yy = np.sum(np.concatenate([np.load('../preprocessor/store_npy_condor/preselected_ggfkl5p6_mc16%s.npy'%m) for m in ['a', 'd', 'e']])['eventweight']) ==> <font color=red>***5.186224164569065***</font>

- Plot the myy distributions after preselections

  - Training Multiclass with four classes: kl=1, kl=0, kl=5.6, k2v=0/3

- ```bash
  # make the multiclass training
  python3 bbyyoptimization.py -c json/multiclassification/four_multiclass_1116.json -o four_multiclass_1116/multiclass --train --bdtscore
  
  # perform the application of the multiclass
  python3 bbyyoptimization.py -c json/multiclassification/four_multiclass_1116.json -o four_multiclass_1116/multiclass -m four_multiclass_1116/multiclass/booster_multiclass_20221116.dat --apply -a store_npy_mbdt
  
  # perform the appltication of the multiclass to get BDT score
  python3 binaryoptimization.py -c json/binaryclassification/devfour_1116/bin_zero.json -o four_multiclass_1116/ -p four_multiclass_1116/multiclass/store_npy_mbdt
  
  # apply the best BDT cut value
  python3 binaryoptimization.py -c json/binaryclassification/devfour_1116/bin_zero.json -o four_multiclass_1116 --applybdt -t four_multiclass_1116
  ```

  HERE: need to think more about it, due to lack of training.

# Updated Procedure of Training with Splitted Sets

- three sets: train, test and valid used to train the multi-labels model as well as binary models
- Structure data inputs for training:

![image-20221126092307087](/Users/shen/Library/Application Support/typora-user-images/image-20221126092307087.png)

- **Multi-labels** training to get model  ==> to label events belonging to which class
- Events split by **eventNumber**: ***public_train_mbdt => 2/3***, Validation: ***public_test => 1***, Test: ***private_test => 0*** 
  - training use two sets: ***public_train,*** ***public_test***
  - apply the model to validation set of these used signal samples
- **Binary classification** to get model of each class
- Events split by **eventNumber**: ***public_train => 0/1***, Validation: ***public_test => 2***, Test: ***private_test => 3*** 
  - training use two sets: ***public_train,*** ***public_test***
  - signal: the events classified to a certain class by multi-label model
  - background: all single Higgs, diphoton and ttyy samples (Without the multi-label model applied)
  - apply all models to public test set and then perform the event categorization (choose the BDT cuts)
- **Apply models** of two layer to all samples
  - for each mc compaign of each process, the sample should be slimmed by training variables due to only those variables considered for the models and then perform the predict to save the classlabels as well as single output scores



- Training Procedures:

```bash
# training with
python3 trainingcondor.py -j json/trainingjson/xgbtraining_KLK2V.json --layer multiclass

python3 bbyyoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --train
python3 bbyyoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --apply
python3 bbyyoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --table  # under batch mode due to no pdflatex on condor sever nodes   # one could change parameter `tableall` for all events or datasets
python3 binaryoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --train --runclass ggFSM
python3 binaryoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --train --runclass VBFSM
python3 binaryoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --train --runclass ggFBSM
python3 binaryoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --train --runclass VBFBSM
python3 binaryoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --apply # apply all training, validation and testing sets for categorizarion
python3 binaryoptimization.py -j json/trainingjson/xgbtraining_SM_BSM.json --categorize # get the best BDT cuts to all samples
python3 applymodels.py -j json/trainingjson/xgbtraining_SM_BSM.json --apply # final application for samples
python3 applymodels.py -j json/trainingjson/xgbtraining_SM_BSM.json --yields # create the yields table after event selection
python3 applymodels.py -j json/trainingjson/xgbtraining_SM_BSM.json --datamc # preselection and bdtselection
python3 postanalysis.py -j json/analysisjson/yields_collect_SM_BSM.json --condor
python3 postanalysis.py -j json/analysisjson/yields_collect_SM_BSM.json --run2D
```

- signal modeling command:

```bash
Running: root -l -q -b "/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/cppmacro/Modelling_bbyy.C(\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/models\",\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/inputs/HH_H_ggFHH_SM_LP_tree.root\",\"ggFHH_SM_LP\",false,false,false,\"HH_H\",\"DSCB\",115,135,3)"
Running: root -l -q -b "/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/cppmacro/Modelling_bbyy.C(\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/models\",\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/inputs/HH_H_ggFHH_SM_HP_tree.root\",\"ggFHH_SM_HP\",false,false,false,\"HH_H\",\"DSCB\",115,135,3)"
Running: root -l -q -b "/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/cppmacro/Modelling_bbyy.C(\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/models\",\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/inputs/HH_H_VBFHH_SM_LP_tree.root\",\"VBFHH_SM_LP\",false,false,false,\"HH_H\",\"DSCB\",115,135,3)"
Running: root -l -q -b "/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/cppmacro/Modelling_bbyy.C(\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/models\",\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/inputs/HH_H_VBFHH_SM_HP_tree.root\",\"VBFHH_SM_HP\",false,false,false,\"HH_H\",\"DSCB\",115,135,3)"
Running: root -l -q -b "/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/cppmacro/Modelling_bbyy.C(\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/models\",\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/inputs/HH_H_ggFHH_BSM_LP_tree.root\",\"ggFHH_BSM_LP\",false,false,false,\"HH_H\",\"DSCB\",115,135,3)"
Running: root -l -q -b "/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/cppmacro/Modelling_bbyy.C(\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/models\",\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/inputs/HH_H_ggFHH_BSM_HP_tree.root\",\"ggFHH_BSM_HP\",false,false,false,\"HH_H\",\"DSCB\",115,135,3)"
Running: root -l -q -b "/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/cppmacro/Modelling_bbyy.C(\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/models\",\"/lustre/collider/shenqiuping/dihiggs/hhyybb-optimization/parameters/inputs/HH_H_VBFHH_BSM_tree.root\",\"VBFHH_BSM\",false,false,false,\"HH_H\",\"DSCB\",115,135,3)"
```

# NEED to Check the BDT cut Module 

- see what the details of the bdt cuts to make sure the BDT cut works well
- Significance, yields and efficiency plots considering 2D and 1D plots

# Objective, 

Slide 5: yy yields scaled to 1.34 to account for 

Slide 7: One sentence and move data set to backup slides and 

Slide 10: Describe the application of multi-lable 


