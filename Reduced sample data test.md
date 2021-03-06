Things always need to check before running
---------------------------------------------------
Because Julia keeps growing in a very fast pace, it is necessary to do the package updating
every time you want to use it in case that some bugs might be caused by the change of some
other packages. If you are not following the master developing version, you need to do

    Pkg.checkout("PhyloNetworks")
    Pkg.update()
    quit()
    
Then open Julia again and 

    using PhyloNetworks
    
To make sure there is not a problem caused by the old version of the package.

    
Run SNaQ on reduced data set with current functions
---------------------------------------------------

input:

- reduced tree file: `data/raxml_1387_sample_5species4alleles.tre`
- map file:  `data/strain2bin_map.csv` (which contains extra info on species that
  are not represented in the reduced trees)
- starting tree: do this within julia

    T = readTopology("(bin0,(bin2a,(bin1a,(bin1b,bin1c))));")

The starting tree needs to contain all the taxa we have in the CFTable file,
which you could check by the function:
    
    PhyloNetworks.unionTaxa(d.quartet)
    
and here we have totally 19 taxa so that the starting tree I choose is 

    T = readTopologyLevel1("((VFI114,VchN16961),((GBAG,((Ddi453,((Dic586,Dpa2511),
    ((BspEniD312,(BsaATCC15712,PectoSCC3193),(PcaPCC21,Pwa43316),
    (WPP1692,SCRI1043)),PcaPC1)),(DzeEch1591,ECH3937)),GCDA)),(Pae1,PstDC3000)));")
    
If you want to make sure that the starting tree has all the taxa (because sometimes
the tree might be really huge), you could use

    tipLabels(T)

to get the full list of taxa of the starting tree.
    
Measure time used for each commend running
---------------------------------------------------

To do the time measuring for each commend we performing on the reduced data set,
the package CPUTime will be used.

    Pkg.clone("https://github.com/schmrlng/CPUTime.jl.git")
    using CPUTime

The commend we want in COUTime package is *tic()* and *toc()*,
which will be used in this way.

    tic();
    ***;
    toc()
    
The *** part will be the commend you want to measure the running time for,
and after toc(), the consuming CPU time for the commend will be estimated.
    
Input data
---------------------------------------------------

The data file used to input here is the tree file `data/raxml_1387_sample_5species4alleles.tre`.

    smallsample = readMultiTopology("raxml_1387_sample_5species4alleles.tre")
    
Then we will create the CFTable file

    d=readTrees2CF("raxml_1387_sample_5species4alleles.tre", CFfile="tableCFall.txt")
    less("tableCFall.txt")
    
However here we have 1387 sample trees inside the input data file,
which means that the CFTable file we generate will have many lines.
Now we want to use the SNaQ package for the tree selection and then
there is one final step for the preparation. We need to convert the
CFTable file in to a DataCF object so that the package could accept.
Here are two ways. One is that we directly convert it and the other
one is that we convert it by two steps. We can check the time these
two ways used here.
The first:

    tic();
    d=readTableCF("tableCFall.txt");
    toc()

The measured time is about 0.133 seconds.
The second:

    using DataFrames;
    tic();
    dat = readtable("tableCFall.txt");
    d = readTableCF(dat);
    toc()

The measured time is about 0.123 seconds.
It seems that there is no big difference, but the original data could be
huge big so at that time, two steps may be better because it is much faster
to convert a dataframe file into the DataCF object we want. One last step is
to have the starting tree for the optimization and the starting file we just
create here: `data/start.txt` 
And to start *SNaQ*, we want the starting tree to be level 1.

    T=readTopology("startT.txt")

Now the preparation is done and we could start with *SNaQ*.

Network Estimation
---------------------------------------------------

Plotting the tree from the basic starting tree.

     tic();  
     p = plot(net1, showGamma=true); 
     toc()
     
elapsed time: 0.019139512 seconds


Time Test
---------------------------------------------------

Though *SNaQ* is an efficient and powerful software for polygenetic tree,
if the dataset is huge, it will just be a time-consuming job. So the reason
we want to do the proper timetests us that we want to find the proper condition
that could be fast enough and guarantee the accuracy of the final results. 

What the time test doing is that we will apply the reduced data sample into *SNaQ*
and do a little change to the default parameter of *SNaQ*. The results of each time
test will be compared to the best tree we have by thoroughly searching without
changing anything. There are three most important things we use to determine whether
the changing parameter could improve the efficiency or not. They are the value
of log-likelihood of the result tree, the taking time for the whole run and the
location of hybridization happening.

Plotting the starting tree after choosing choosing by *SNaQ* with *hmax* equals to 0. 

 hmax = 1,
 tolerance parameters: ftolRel=1.0e-5, ftolAbs=1.0e-6, xtolAbs=0.0001, xtolRel=0.001.
 max number of failed proposals = 100, multiplier M = 10000.



The starting tree we used is just set by ourself and generated by
*readTopologylevel1*, which means that there is no any selection progress
for the starting tree. As a result, when we use this tree to run *snaq!*
it will be inevitable to find a propor tree first and then do the model selection.
So the there is a lot of time waste. To save the time, we may want to find a
better starting tree at the very beginning. The method to do that is to run
*snaq!* with *hmax* equal to 0 and *runs* equal to 1 at first. The *hmax* equal to 0
means that we just want to select a single tree here and the reason we only
need *runs* equal to 1 is that all whatever seed we use, the final result of the
this single tree will always be the same. Then we just use the best starting tree
result to start the selection process with *hmax* equal to 1.
The starting tree result file is saved with name `bT1`:

    bT1 = snaq!(T,d, hmax = 0, runs = 1, filename = "bT1_snaq")

- seed 66077 for run 1  Begin: 11:19:05  End: 14:07:10  loglik of best 3917.962149687413
It takes about 50 minutes to find the best starting tree.

The *snaq!* function will automatically generate three files to save the information
of running errors, the running process and the running result. We can just use
*readTopology* to read the best starting tree directly from the file `bT1.out`. 

    T = readTopology("bT1.out")

Then use this best starting tree we could do *snaq!* again to check the consuming time
and the precision of the final selection. The result file is saved with name `newtry1`:

    newtry1 = snaq!(T,d, filename = "newtry1_snaq")

- seed: 36252 for run 1  Begin: 16:06:28 loglik of best  3441.0949164040185
- seed: 92678 for run 2  Begin: 19:52:19 loglik of best  3799.6169101747423
- seed: 3426 for run 3   Begin: 21:57:18 loglik of best  3860.3260109935527
- seed: 30204 for run 4  Begin: 00:33:57 loglik of best  3798.5435427302073
- seed: 14351 for run 5  Begin: 02:33:02 loglik of best  3439.745415257459
- seed: 76736 for run 6  Begin: 05:53:14 loglik of best  3860.447696910964
- seed: 41383 for run 7  Begin: 07:30:52 loglik of best  3443.9592402404087
- seed: 10404 for run 8  Begin: 10:04:39 loglik of best  3913.542411314058
- seed: 12725 for run 9  Begin: 11:43:34 loglik of best  3861.2845423538033
- seed: 37357 for run 10 Begin: 14:00:59 loglik of best  3439.9405185286623
- elapsed time: 88579.306341032 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.39163655768885497,(Dpa2511,(((BsaATCC15712,BspEniD312):0.312971054428727,((PcaPC1,(PcaPCC21,WPP1692):1.143680121186059):0.08613154547826318,(SCRI1043,((PectoSCC3193,Pwa43316):0.7567694553681763,#H20:5.359286245393218::0.021690559755770736):0.20334881976891134):0.43865294980613584):2.4960576373683816):1.4052192932437961,(((GCDA,GBAG):2.219203542646777,((Pae1,PstDC3000):3.5601328104494443,(VFI114,VchN16961):2.518302625246832):1.313552428201745):0.38506752249319154)#H20:1.1442637384850836::0.9783094402442293):1.711847507984087):1.5069797845967137):0.7567109208877364); 
with -loglik 3439.745415257459

The running time for the model selection is about 24 hours and considering that we
spend about 1 hour to wait for the best starting tree, we need about 25 hours to get
the best selected model with all the default parameters. This best selected model
has the log-likelihood equal to 3439.75, which is better than the model from the
starting tree we set by ourself. This model will be used to determine the precision
of the following timetest after changing different parameters.


Maximum Number of Failed Proposals
---------------------------------------------------

The parameter used here is *Nfail*, which represents the meaning that the maximum
number of times that new topologies are proposed and rejected (in a row). The default
number of this parameter is 100. However, the default number is chosen to face
significantly large dataset. In another word, we may choose a smaller number for
the specific dataset by ourself to reduce the time it takes. As a result of reducing 
the repeating time, the time we need will consequently decrease.
The only thing we want to know is how precisely the function will perform. 

Max number of failed proposals = 10

The result file is saved with name `timetest1`:

    timetest1 = snaq!(T,d, Nfail = 10, filename = "temtest1_snaq")

- seed: 30312 for run 1  Begin: 16:35:20 loglik of best 3455.845945853218
- seed: 1440 for run 2   Begin: 17:47:11 loglik of best 3860.533441026289
- seed: 98511 for run 3  Begin: 18:04:50 loglik of best 3859.115780423511
- seed: 80657 for run 4  Begin: 18:29:03 loglik of best 3880.752407454445
- seed: 78568 for run 5  Begin: 18:52:02 loglik of best 3811.554199979079
- seed: 63245 for run 6  Begin: 19:11:35 loglik of best 3777.823939751652
- seed: 81138 for run 7  Begin: 19:49:40 loglik of best 3443.958828187268
- seed: 89472 for run 8  Begin: 20:16:18 loglik of best 4357.230840522077
- seed: 89680 for run 9  Begin: 20:38:41 loglik of best 3465.6935439833105
- seed: 34599 for run 10 Begin: 20:54:05 loglik of best 3892.1805493222896
- elapsed time: 16688.01510346 seconds

MaxNet is(ECH3937,Ddi453,((Dic586,DzeEch1591):0.39151452022020755,(Dpa2511,(((BsaATCC15712,BspEniD312):0.31165264651122426,((PcaPC1,(PcaPCC21,WPP1692):1.144025891196133):0.0876790364165445,(((PectoSCC3193,Pwa43316):0.9529701731377197,SCRI1043):0.0632647451890301,#H20:8.251812487790279e-7::0.023694403794416646):0.37365021164620515):2.5071408957817876):1.4168426104118295,(((GCDA,GBAG):2.21912632231476,((Pae1,PstDC3000):3.565361799130726,(VFI114,VchN16961):2.518277890640175):1.3136037135425378):0.42011126229987183)#H20:1.1184877095177088::0.9763055962055833):1.7090991861969702):1.5074073219469928):0.7563231659538783)
with -loglik = 3443.958828187268

When the *Nfail* is 10, the running process is very fast.
It could seen that for most of these 10 runs each just takes about 20 to 30 minutes.
The first run more than 1 hour to finish. Though it does not give the best estimation
among the ten runs, run 1 has the second best log-likelihood of all ten results.
Among all the results, only two models have the log-likelihood lower than 3460 and
the best model only have log-likelihood about 3443.96, which seems nice but may be just
lucky for this run. As a result, only having 10 failed proposals is not enough
to keep the precision as much as possible. The error is still big.

Max number of failed proposals = 25

The result file is saved with name `timetest2`:

    timetest2 = snaq!(T,d, Nfail = 25, filename = "temtest2_snaq")

- seed: 28669 for run 1  Begin: 22:31:46 loglik of best 3445.565423084598
- seed: 18251 for run 2  Begin: 23:15:13 loglik of best 3495.1967449792023
- seed: 17055 for run 3  Begin: 00:11:08 loglik of best 3821.192476733005
- seed: 37382 for run 4  Begin: 00:38:31 loglik of best 3789.321048313865
- seed: 54803 for run 5  Begin: 01:34:25 loglik of best 3880.879215672171
- seed: 44596 for run 6  Begin: 02:32:10 loglik of best 3477.284003600159
- seed: 77913 for run 7  Begin: 03:06:49 loglik of best 4663.104143504624
- seed: 56320 for run 8  Begin: 04:04:42 loglik of best 3456.376434209783
- seed: 84641 for run 9  Begin: 05:50:21 loglik of best 3452.7128180499426
- seed: 32265 for run 10 Begin: 07:28:32 loglik of best 3452.7428754539405
- elapsed time: 37137.96354747 seconds

MaxNet is (ECH3937,Ddi453,(((((BsaATCC15712,BspEniD312):0.31216862726697664,((PcaPC1,(PcaPCC21,WPP1692):1.1475272878942944):0.08845813076320641,(((PectoSCC3193,Pwa43316):0.9519692222828424,SCRI1043):0.14365203312222513,#H20:0.00033392625089012976::0.02436318909483342):0.2894145898441131):2.512137952982982):1.4185616701110089,(((GCDA,GBAG):2.219171637809479,((Pae1,PstDC3000):3.574974826067273,(VFI114,VchN16961):2.519542917209923):1.3151513661138128):0.38415622190444954)#H20:1.1685684167616137::0.9756368109051666):1.7110372843444337,Dpa2511):1.50423257630288,(Dic586,DzeEch1591):0.39180926161717833):0.758082798846433); 
with -loglik 3445.565423084598

Here *Nfail* is increased to be 25. The time consumed this time is more than twice
of the time used when *Nfail* is 10. However, the result is just far better.
Among all the ten runs, 4 of them have the log-likelihood lower than 3460, and six
of these runs have the log-likelihood are lower than 3500. It means that we only need
about 10 hours to get a highly possibly accurate result. However, here *Nfail* equal
to 25 could only give us a model with log-likelihood equal to 3445.57 as the best of
ten, which is not close enough to the best model.  

Because the seed we use to get the best tree model is 14351, we may wonder that
if we set the same seed for max 25 failed proposals, will the model chosen here
be close enough to the best one. The result file is saved with name `timetest6`:

    timetest6 = snaq!(T,d,Nfail=25,seed=14351,filename="timetest6_snaq")

- seed: 14351 for run 1  Begin: 01:42:33 loglik of best 3439.831919169519
- seed: 90844 for run 2  Begin: 03:04:53 loglik of best 3574.6315802939253
- seed: 78295 for run 3  Begin: 04:20:32 loglik of best 3638.6980296735205
- seed: 76778 for run 4  Begin: 05:00:36 loglik of best 3439.871728209358
- seed: 79382 for run 5  Begin: 06:29:13 loglik of best 3864.2804247681347
- seed: 13085 for run 6  Begin: 07:30:51 loglik of best 3468.763567622874
- seed: 31758 for run 7  Begin: 08:57:55 loglik of best 3439.9256526546883
- seed: 82679 for run 8  Begin: 09:29:24 loglik of best 3778.554994417975
- seed: 9362 for run 9   Begin: 09:58:15 loglik of best 3477.222505317204
- seed: 15462 for run 10 Begin: 10:53:03 loglik of best 3450.417959586848
- elapsed time: 39287.796202476 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.3915743696751431,(Dpa2511,(((BsaATCC15712,BspEniD312):0.3129962171814515,((PcaPC1,(PcaPCC21,WPP1692):1.143315354343887):0.08633693067081614,(SCRI1043,((PectoSCC3193,Pwa43316):0.7527887542169304,#H20:0.3142796702367138::0.021638165529126202):0.2074329101735221):0.4386180880824666):2.495505518400126):1.4049146946768694,(((GCDA,GBAG):2.21913648810776,((Pae1,PstDC3000):3.560597558863374,(VFI114,VchN16961):2.5180161266556365):1.3135079823952018):0.3877819667050952)#H20:1.1412673110544422::0.9783618344708738):1.7119821223052967):1.507014138610826):0.7566679598911226); 
with -loglik 3439.831919169519

Here the running time is a bit longer than `timetest2`. We do have a result close
enough to the best model we have (only 0.08 higher in log-likelihood).
Though from this result we know that it is possible that if we try another time, 
there might be some better results by different seeds, it is still too risky to 
just use *Nfail* equal to 25.

Relative and Absolute Differences of the Network score
---------------------------------------------------

The parameters used here are *ftolRel* and *ftolAbs*. These two parameters
represent the relative and absolute differences of the network score between
the current and proposed parameters. It is just like one person is climbing
mountain and the method we determine whether or not the person reach the peak
is that we compare the new step and the last step and find the small enough
difference. The two parameters here are just the threshold of such the small
difference. Therefore, if the threshold is bigger, it will be faster to determine
the "peak" we want. The result, however, may not be accurate because there are
better results ignored because of the relatively large threshold. 

ftolRel=0.1, ftolAbs=0.1

The result file is saved with name `timetest3`:

    timetest3 = snaq!(T,d,ftolAbs = 0.1, ftolRel = 0.1, filename = "timetest3_snaq")

- seed: 66086 for run 1  Begin: 14:27:03 loglik of best 3917.962149548205
- seed: 93640 for run 2  Begin: 14:40:59 loglik of best 3917.962152057165
- seed: 73042 for run 3  Begin: 15:04:37 loglik of best 3917.962149548205
- seed: 84445 for run 4  Begin: 15:19:33 loglik of best 3772.173411217855
- seed: 49744 for run 5  Begin: 15:54:27 loglik of best 3917.962149548205
- seed: 86464 for run 6  Begin: 16:09:07 loglik of best 3579.9085278999314
- seed: 94389 for run 7  Begin: 16:46:26 loglik of best 3951.3614867607225
- seed: 5656 for run 8   Begin: 17:00:49 loglik of best 4900.832095524829
- seed: 81047 for run 9  Begin: 17:15:29 loglik of best 3880.409481661765
- seed: 67230 for run 10 Begin: 17:45:09 loglik of best 3917.962149548205
- elapsed time: 12630.994448551 seconds

MaxNet is (GCDA,GBAG,(((Dpa2511,((ECH3937,Ddi453):0.7567332925080691,(Dic586,DzeEch1591):0.39166727385097627):1.5072281487611878):1.732128262799997,((BsaATCC15712,BspEniD312):0.32515119812196397,(((PectoSCC3193,(Pwa43316,#H20:1.2730160030904467::0.012173533618828777):0.0):0.965848365453346,SCRI1043):0.43364838185982363,(PcaPC1,(PcaPCC21,WPP1692):1.1437260053856166):0.08835461160756057):2.2930295593152437):1.3047675710667652):1.445708320415149,(((Pae1,PstDC3000):3.563500986437131,(VFI114,VchN16961):2.5209405814127477):0.3178165230655552)#H20:1.1781499900740195::0.9878264663811712):2.1883558640518825); 
with -loglik 3579.9085278999314

The total running time is 3 hours and an half. The result is really inaccurate. Most of
these runs have the log-likelihood higher than 3900 and the best one just have
3579.91, which is till far away from 3439. 0.1 is still a threshold too wide for
the model selection. One thing, however, we can learn from it is that to change
the threshold of *ftolRel* and *ftolAbs* may have huge influence on the consuming time.
 
ftolRel=0.01, ftolAbs=0.01

The result file is saved with name `timetest4`:

    timetest4 = snaq!(T,d,ftolAbs = 0.01, ftolRel = 0.01, filename = "timetest4_snaq")

- seed: 62366 for run 1  Begin: 19:20:33 loglik of best 3471.5565201273243
- seed: 30970 for run 2  Begin: 20:19:19 loglik of best 3703.723965572843
- seed: 50958 for run 3  Begin: 20:58:29 loglik of best 3477.222812511022
- seed: 69102 for run 4  Begin: 22:01:41 loglik of best 3540.174570557619
- seed: 88420 for run 5  Begin: 22:35:10 loglik of best 3880.260875679105
- seed: 17226 for run 6  Begin: 23:04:08 loglik of best 3917.9621494886715
- seed: 86909 for run 7  Begin: 23:27:43 loglik of best 3637.6350082096656
- seed: 17807 for run 8  Begin: 23:55:16 loglik of best 3642.4785098371476
- seed: 13572 for run 9  Begin: 00:24:10 loglik of best 4256.274779596646
- seed: 91017 for run 10 Begin: 00:57:57 loglik of best 3732.0899076554983
- elapsed time: 21942.346502542 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.391642176389816,(Dpa2511,(((BsaATCC15712,BspEniD312):0.31001301816573235,(((PectoSCC3193,Pwa43316):0.9526387747495383,SCRI1043):0.4267130120588206,((PcaPC1,(PcaPCC21,WPP1692):1.145216228713851):0.09291755474087214,#H20:0.02884909732731804::0.027371350938236238):0.0):2.501979205318754):1.4333616710907768,(((GCDA,GBAG):2.2192344480745247,((Pae1,PstDC3000):3.560280116560067,(VFI114,VchN16961):2.518310675198813):1.3135668178648205):0.4510663918933101)#H20:1.1076673374707677::0.9726286490617637):1.7048467025609013):1.5069044285895976):0.7566904051328839); 
with -loglik 3471.5565201273243

When we set the threshold from 0.1 to 0.01, it takes about 2 more hours and the result
is just turns from really-far-away from the best model to far-away from the best model.
There are still 7 out of 10 results have log-likelihood over 3600. Only two results are
lower than 3500. The threshold is still need to be restrained.

ftolRel=0.005, ftolAbs=0.005

The result file is saved with name `timetest5`:

    timetest5 = snaq!(T,d,ftolAbs = 0.005, ftolRel = 0.005, filename = "timetest5_snaq")

- seed: 3888 for run 1   Begin: 02:04:17 loglik of best 3587.0296712389422
- seed: 32306 for run 2  Begin: 02:45:20 loglik of best 3691.1667214936065
- seed: 77486 for run 3  Begin: 03:21:36 loglik of best 3873.9757750437807
- seed: 20847 for run 4  Begin: 03:53:23 loglik of best 3914.1585393577293
- seed: 79581 for run 5  Begin: 04:37:40 loglik of best 3452.1660386386097
- seed: 25572 for run 6  Begin: 05:13:03 loglik of best 3443.9379207438174
- seed: 67336 for run 7  Begin: 06:15:29 loglik of best 3589.270148242795
- seed: 49916 for run 8  Begin: 06:48:45 loglik of best 3864.3664078535126
- seed: 1755 for run 9   Begin: 07:25:48 loglik of best 3537.7516097086072
- seed: 97345 for run 10 Begin: 08:06:10 loglik of best 3471.417434728496
- elapsed time: 23949.375026384 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.3918919971666905,(Dpa2511,(((BspEniD312,BsaATCC15712):0.3120159748027607,((PcaPC1,(PcaPCC21,WPP1692):1.1434580441700692):0.0864065512014871,((PectoSCC3193,Pwa43316):0.955526786889747,(SCRI1043,#H20:0.6514914098372533::0.02296772745186695):0.0):0.4379592134502702):2.5031669842366586):1.4129721082992313,(((GCDA,GBAG):2.220661039387234,((Pae1,PstDC3000):3.5628664333252815,(VFI114,VchN16961):2.516544766284679):1.3138706371578928):0.4054335787714331)#H20:1.1302001347489872::0.9770322725481331):1.7104398706169768):1.506535828817771):0.7549551377319391); 
with -loglik 3443.9379207438174

It is almost the same situation, the time consumed does not changed a lot. The precision
could not be guaranteed. 4 out of 10 results have log-likelihood ove 3600 and only 3
results are lower than 3450. The best result is still not close enough to the best model.

Because the seed we use to get the best tree model is 14351, we may wonder that
if we set the same seed for *ftolRel* and *ftolAbs* both equal to 0.005, will it be
possible that model chosen here can be close enough to the best one.
The result file is saved with name `timetest7`:

    timetest7 = snaq!(T,d,ftolAbs = 0.005, ftolRel = 0.005,seed=14351, filename = "timetest7_snaq")

- seed: 14351 for run 1  Begin: 14:49:36 loglik of best 3446.784528843177
- seed: 90844 for run 2  Begin: 15:33:03 loglik of best 3686.9890626963256
- seed: 78295 for run 3  Begin: 16:23:03 loglik of best 3443.339598175505
- seed: 76778 for run 4  Begin: 17:16:34 loglik of best 3442.2165134307375
- seed: 79382 for run 5  Begin: 17:55:45 loglik of best 3649.408096666195
- seed: 13085 for run 6  Begin: 18:43:50 loglik of best 3929.196565135153
- seed: 31758 for run 7  Begin: 19:19:44 loglik of best 3443.847375485665
- seed: 82679 for run 8  Begin: 19:59:24 loglik of best 3647.4898602407634
- seed: 9362 for run 9   Begin: 20:53:52 loglik of best 3443.7578723938473
- seed: 15462 for run 10 Begin: 22:36:34 loglik of best 3745.650670766903
- elapsed time: 29822.147601027 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.3983580757962606,(Dpa2511,(((BsaATCC15712,BspEniD312):0.31311563379272006,((SCRI1043,((PectoSCC3193,Pwa43316):0.7434145351324302,#H20:9.521148539855115::0.02275145365910982):0.21880952509738338):0.43987509020656956,(PcaPC1,(PcaPCC21,WPP1692):1.1415334084093285):0.08717494852851908):2.5063632661480293):1.4091395201224555,(((GBAG,GCDA):2.221820889562346,((Pae1,PstDC3000):3.5231220173541926,(VFI114,VchN16961):2.5194507206652554):1.3143751192325326):0.5342003221666584)#H20:0.9659415766741903::0.9772485463408902):1.7105228789930695):1.5059423283907745):0.7593455611813633); 
with -loglik 3442.2165134307375

After setting the seed, the time used increased much but the best result is only
a little more accurate than before. Further more, the best of these ten runs do not
exist at seed 14351, which means that the threshold is still to large for the
optimization.

Relative and Absolute Differences
---------------------------------------------------

*xtolAbs* and *xotRel* are parameters used to decide relative and absolute differences between 
the current and proposed parameters. The effects are similar to *ftolRel* and *ftolAbs*.
So it is the same to find a suitable threshold reducing the time while maintaining the accuracy at most.

xtolAbs=0.001, xtolRel=0.1
The result file is saved with name `timetest8`:

    timetest8 = snaq!(T,d,xtolAbs = 0.001,xtolRel=0.1,filename="timetest8_snaq")

- seed: 15989 for run 1  Begin: 23:57:35 loglik of best 3452.3876458806562
- seed: 6984 for run 2   Begin: 01:12:20 loglik of best 3649.445418523897
- seed: 38587 for run 3  Begin: 02:11:42 loglik of best 3777.9352821695843
- seed: 99651 for run 4  Begin: 03:00:49 loglik of best 3442.439961291139
- seed: 64228 for run 5  Begin: 04:49:35 loglik of best 3513.9161425637476
- seed: 74307 for run 6  Begin: 05:38:05 loglik of best 3789.376218709045
- seed: 43687 for run 7  Begin: 07:48:58 loglik of best 3826.9900332206757
- seed: 1715 for run 8   Begin: 10:13:35 loglik of best 3866.7870677668575
- seed: 96935 for run 9  Begin: 10:50:43 loglik of best 3579.7205355547767
- seed: 68739 for run 10 Begin: 13:42:33 loglik of best 3439.8619471395227
- elapsed time: 51589.342317181 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.3916766725985431,(Dpa2511,(((BsaATCC15712,BspEniD312):0.31294725765853804,((PcaPC1,(PcaPCC21,WPP1692):1.1437156931908767):0.08608746836562556,(SCRI1043,((PectoSCC3193,Pwa43316):0.7543309228495015,#H20:0.0::0.021662238807581512):0.20586829315661892):0.43868703331053177):2.495794290042634):1.4050071204873744,((((Pae1,PstDC3000):3.560193488161601,(VFI114,VchN16961):2.518337458040133):1.3135561489799559,(GCDA,GBAG):2.2192448213792813):0.3923113840252171)#H20:1.1361267214332125::0.9783377611924184):1.7118802770698962):1.5069633090207561):0.7566893744145432); 
with -loglik 3439.8619471395227

We only run one turn for *xtolAbs* and *xotRel* because the result we get is really
close to 3439.75. And this result is not even under seed 14351. It is just from another
random seed chosen automatically. It takes about 14 hours to finish all the process.
Considering the 25 hours we need for the best model and the very small difference
between this model and the best one, I should say this is a successful trade-off.
Another thing we may learn from this result is that *ftolRel* and *ftolAbs* have
huge influence on both time consuming and the precision. However, *xtolAbs* and *xotRel*
do not should many direct affects.

Timetest Combinations 
---------------------------------------------------

ftolRel=0.0001, ftolAbs=1.0e-5, max number of failed proposals = 50

    timetest9 = snaq!(T,d,Nfail = 50, ftolAbs = 0.0001, ftolRel = 0.00001, filename = "timetest9_snaq")

- seed: 45123 for run 1  Begin: 23:48:49 loglik of best 3815.545968256916
- seed: 49019 for run 2  Begin: 01:18:10 loglik of best 3787.0605586031566
- seed: 76390 for run 3  Begin: 02:30:21 loglik of best 3791.4926928638743
- seed: 14769 for run 4  Begin: 03:21:12 loglik of best 4150.301140220645
- seed: 86576 for run 5  Begin: 04:17:45 loglik of best 3801.451960188641
- seed: 93796 for run 6  Begin: 05:09:41 loglik of best 3798.50943549043
- seed: 14870 for run 7  Begin: 06:07:11 loglik of best 3915.8939408518277
- seed: 411 for run 8    Begin: 06:27:22 loglik of best 3827.385781272363
- seed: 97830 for run 9  Begin: 07:26:28 loglik of best 3450.3233016115105
- seed: 69666 for run 10 Begin: 07:56:00 loglik of best 3778.7105185065047
- elapsed time: 34831.465925074 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.39170493794983896,(Dpa2511,((((SCRI1043,((PectoSCC3193,Pwa43316):0.12695669174600074,#H20:0.0003860917252302107::0.01752221317590075):0.8395319744676957):0.44553511415037067,(PcaPC1,(PcaPCC21,WPP1692):1.1432380456799685):0.08418990983178257):2.451438453022471,(BsaATCC15712,BspEniD312):0.31610516138559847):1.3792496742027545,((((Pae1,PstDC3000):3.56158819342393,(VFI114,VchN16961):2.5191151472528843):1.3135857868565666,(GCDA,GBAG):2.219023037697311):0.2749161925916056)#H20:1.2444354590832252::0.9824777868240993):1.7146259519309166):1.506800694807227):0.7555289107768889); 
with -loglik 3450.3233016115105

This combination could be treated as a failed one and this failure is predictable.
Considering what we learnt previously, we may still need to have a smaller threshold
for *ftolRel* and *ftolAbs*. Here *Nfail* is only 50, the result just gets worse
consequently. 

ftolRel=1.0e-5, ftolAbs=5.0e-6, max number of failed proposals = 50
The result file is saved with name `timetest11`:

    timetest11 = snaq!(T,d,Nfail = 50, ftolAbs = 0.000005, ftolRel = 0.00001, filename = "timetest11_snaq")

- seed: 25765 for run 1  Begin: 10:14:44 loglik of best 3910.2358532170542
- seed: 72642 for run 2  Begin: 10:42:22 loglik of best 3792.860240805273
- seed: 71118 for run 3  Begin: 11:55:25 loglik of best 4744.287404792135
- seed: 60193 for run 4  Begin: 12:11:22 loglik of best 3463.518459552619
- seed: 6044 for run 5   Begin: 13:02:53 loglik of best 3760.2008830487707
- seed: 50574 for run 6  Begin: 13:42:55 loglik of best 3513.1123514185306
- seed: 40432 for run 7  Begin: 14:31:28 loglik of best 3463.469701449863
- seed: 78407 for run 8  Begin: 15:46:21 loglik of best 3470.554295934314
- seed: 36628 for run 9  Begin: 16:41:52 loglik of best 3462.385160724953
- seed: 16176 for run 10 Begin: 17:45:54 loglik of best 3879.77242509607
- elapsed time: 67926.502059791 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.39155631541707664,((((BsaATCC15712,BspEniD312):0.31217927705976223,((PcaPC1,(WPP1692,PcaPCC21):1.1438091322521673):0.08712145893940737,((PectoSCC3193,Pwa43316):0.9543426762582049,(SCRI1043,#H20:0.007605993623044121::0.022998304749172777):1.9816595520948545e-5):0.43779737351555964):2.5039539618405255):1.4128254168206718,(((GCDA,GBAG):2.2190383813150985,((PstDC3000,Pae1):3.5613836868963227,(VFI114,VchN16961):2.5182981608124035):1.3135712023542436):0.41514739382038435)#H20:1.1187092819268454::0.9770016952508273):1.710149202795123,Dpa2511):1.5070011334129565):0.7566133704279334); 
with -loglik 3444.0098186152277

The result of this test is still not good enough. Though the threshold for
*ftolRel* and *ftolAbs* has been strained again, *Nfail* equal to 50 may still be
too small. To improve it, it might be one ideal that we just increase *Nfail*.
However, the time is an important factor. Here it has already taken round to 19 hours,
which does not have a strong effect in reducing the time consuming.
It's a bad trade-off.

xtolAbs=0.01, xtolRel=0.1, max number of failed proposals = 50
The result file is saved with name `timetest12`:

    timetest12 = snaq!(T,d, Nfail = 50, xtolAbs = 0.01, xtolRel = 0.1, filename = "timetest12_snaq")

- seed: 39416 for run 1  Begin: 00:31:39 loglik of best 3900.8901641074967
- seed: 25304 for run 2  Begin: 00:57:03 loglik of best 3888.0041653809676
- seed: 91050 for run 3  Begin: 01:14:43 loglik of best 3466.4105213575117
- seed: 33576 for run 4  Begin: 02:45:14 loglik of best 3824.438708542144
- seed: 80968 for run 5  Begin: 03:03:02 loglik of best 3799.468776782879
- seed: 27540 for run 6  Begin: 03:36:54 loglik of best 3452.179386924025
- seed: 71284 for run 7  Begin: 04:09:19 loglik of best 3637.7169579613746
- seed: 74437 for run 8  Begin: 04:33:10 loglik of best 3452.1753017846327
- seed: 83113 for run 9  Begin: 05:03:00 loglik of best 3452.21341511851
- seed: 46295 for run 10 Begin: 05:26:08 loglik of best 3452.687459026667
- elapsed time: about 19620 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.3916433408872647,(Dpa2511,(((BsaATCC15712,BspEniD312):0.3168336042332628,(((Pwa43316,(PectoSCC3193,#H20:0.0016588062403492198::0.01649996065928392):0.10052325901227072):0.9783427497685588,SCRI1043):0.44421758741969736,(PcaPC1,(PcaPCC21,WPP1692):1.1431045256679693):0.08496260948739474):2.438103836585571):1.3719850572015384,(((GCDA,GBAG):2.219170971872821,((Pae1,PstDC3000):3.560583845510604,(VFI114,VchN16961):2.5182215444251157):1.313528231809789):0.24173024635873708)#H20:1.2730643170814233::0.9835000393407161):1.718809217554148):1.5070541600249934):0.7566294420794621); 
with -loglik 3452.1753017846327

It's true that the result from changing *xtolAbs* and *xotRel* independently is
impressive and changing *Nfail* together just reduce huge amount of time.
But the precision could not be retained any more. We could find some clues from
`timetest8` that there are only 2 results having log-likelihood under 3450 and
many others having really high log-likelihood. It is not really wise to reduce
*Nfail*, *xtolAbs* and *xotRel* together.

ftolRel=1.0e-5, ftolAbs=1.0e-5, xtolAbs=0.01, xtolRel=0.1
The result file is saved with name `timetest13`:

    timetest13 = snaq!(T,d, ftolAbs = 0.00001, ftolRel = 0.00001, xtolAbs = 0.01, xtolRel = 0.1, filename = "timetest13_snaq")
 
- seed: 38112 for run 1  Begin: 10:23:58 loglik of best 3452.170110138604
- seed: 89788 for run 2  Begin: 11:26:54 loglik of best 16777.118686159545
- seed: 39728 for run 3  Begin: 12:12:22 loglik of best 3876.993280941909
- seed: 32730 for run 4  Begin: 12:47:02 loglik of best 3730.520478967291
- seed: 3618 for run 5   Begin: 14:59:47 loglik of best 3890.989893184003
- seed: 99237 for run 6  Begin: 15:36:08 loglik of best 3839.352586001547
- seed: 82736 for run 7  Begin: 16:09:56 loglik of best 3439.7857353253703
- seed: 10900 for run 8  Begin: 17:02:07 loglik of best 3890.2504624396247
- seed: 24399 for run 9  Begin: 17:28:29 loglik of best 3452.38426859734
- seed: 29767 for run 10 Begin: 18:21:43 loglik of best 3576.3872647110106
- elapsed time: 31456.993676184 seconds

MaxNet is (ECH3937,Ddi453,((Dic586,DzeEch1591):0.3913807913672653,(Dpa2511,(((BsaATCC15712,BspEniD312):0.3127965530017362,(((PcaPCC21,WPP1692):1.143501603410818,PcaPC1):0.08617791573326726,(SCRI1043,((PectoSCC3193,Pwa43316):0.7616050728197059,#H20:1.1110053383035756::0.021720079595650442):0.19876242906141212):0.4384708547512707):2.4963536617910767):1.4053436772117012,((((Pae1,PstDC3000):3.559882152509993,(VFI114,VchN16961):2.518755914350536):1.3133612219177495,(GCDA,GBAG):2.219616559746328):0.3888544626534135)#H20:1.1400542254183552::0.9782799204043495):1.7117638924346683):1.5069634010734612):0.7566294928373016); 
with -loglik 3439.7857353253703

This is a surprising result. The *ftolRel* and *ftolAbs* here is a little bigger than
`timetest11`, and *xtolAbs* and *xotRel* is also bigger. But the result is a little
better than `timetest8`. The possible reason is that it's just a luck because this random
seed just shows a better selection (there is one with log-lik 16777 which is nonsense)
or under *Nfail* equal to 100, *ftolRel* equal to 0.00001 is just small enough
for this reduced dataset to help select a decent model.
It might be that I'm just lucky enough to have this result, but considering it
only takes fewer than 9 hours, we can do give a try.

Final Comparison of All Results Close Enough to the Best Model
--------------------------------------------------

From all the timetests above, finally `timetest8` and `timetest13` show some nice results. However,
the preivous timetests are all based on the criteria of choosing the log-likelihood of the
selected model as small as possible. However, having a small log-likelihood is not enough
to determine the precision of models from timetests. We also want the models from timetests
to have the same location of hyperdizations as the best model has. To check the hyperdization
locations, we need to visualize the tree model. We will need the function *plot* in
*PhyloNetworks*. So we have three tree models to visualize: `net1_snaq.out`, `timetest8_snaq.out` and
`timetest13_snaq,out`.

    T1 = readTopology("net1_snaq.out");
    T2 = readTopology("timetest8_snaq.out");
    T3 = readTopology("timetest13_snaq.out");
    p1 = plot(T1, showGamma=true)
    p2 = plot(T2, showGamma=true)
    p3 = plot(T3, showGamma=true)

By visualizing these three models, we could find that `timetest8_snaq.out` and `timetest13_snaq,out`
have almost the identical model and the hyperdization happened at the same place with the same probability.
Actully, compared to the best model, these two timetests just have the hyperdization at the same location.
The only difference is the probability of this hyperdization. So probably, these two changes of parameters
could help save the time and retain the precision in this case. `timetest8_snaq.out` needs more than 13 hours
and `timetest13_snaq,out` needs lower than 9 hours. However, generally considering all results of ten runs,
`timetest8_snaq.out` will indeed be better than `timetest13_snaq,out` (because a smaller threshold for
*ftolRel*).

Suggestions
---------------------------------------------------

Running polygentic trees model is definitely a time-consuming process. For some scientists, saving time is
just sving money, so they want a nice answer quickly, which means they persue the efficiency. For some other
scientists, they may had already spent years to collcted the data, so the time is not the burden for them to
find the most accurate model. Then they just want precision.
From the tests to this reduced sample dataset of polygentic trees, my suggestions is that when using *snaq!* to
run the model selection, first you may need to use *hmax* equal to 0 to find the best starting tree, and Second,
you may just increase the threshold of parameters  *xtolAbs* and *xotRel* to have a general idea about the whole
model in less time. Then persuing the efficiency or persuing the precision just depends on the users themselves.
