written by: Nutchaya Phumekham, July, 2022

### This is a re-write of the [condor raw data analysis](https://github.com/nutty7fold/cern-crab-data-analysis/blob/main/issue7312/final-brief-analysis-condor-raw-data.ipynb) in order to show example usage of the [crab analysis guide](https://github.com/nutty7fold/cern-crab-data-analysis/blob/main/crab_data_analysis_doc/analysis_guide.md). 

#### Important Note: save file [utils.py](https://github.com/nutty7fold/cern-crab-data-analysis/blob/main/crab_data_analysis_doc/utils.py) in the same directory as your current Jupiter Notebook to use functions from it.


```python
from utils import (
    _to_dict,
    _donut,
    _pie,
    _line_graph,
    _other_fields,
    _exitcode_info,
    _better_label
)
from datetime import datetime, date, timedelta
from pyspark.sql.functions import (
    col,
    lit,
    when,
    sum as _sum,
    count as _count,
    first,
    date_format,
    from_unixtime
)
import numpy as np
import pandas as pd
from pyspark.sql.types import (
    StructType,
    LongType,
    StringType,
    StructField,
    DoubleType,
    IntegerType,
)
```


```python
def _get_schema():
    return StructType(
        [
            StructField(
                "data",
                StructType(
                    [
                        StructField("CMSSite", StringType(), nullable=True),
                        StructField("RecordTime", LongType(), nullable=False),
                        StructField("InputData", StringType(), nullable=True),
                        StructField("CMSPrimaryDataTier", StringType(), nullable=True),
                        StructField("Status", StringType(), nullable=True),
                        StructField("OverflowType", StringType(), nullable=True),
                        StructField("WallClockHr", DoubleType(), nullable=True),
                        StructField("CoreHr", DoubleType(), nullable=True),
                        StructField("CpuTimeHr", DoubleType(), nullable=True),
                        StructField("RequestCpus", LongType(), nullable=True),
                        StructField("Type", StringType(), nullable=True),
                        StructField("CRAB_DataBlock", StringType(), nullable=True),
                        StructField("GlobalJobId", StringType(), nullable=False),
                        StructField("ExitCode", LongType(), nullable=True),
                        StructField("Chirp_CRAB3_Job_ExitCode", LongType(), nullable=True),
                        StructField("Chirp_WMCore_cmsRun_ExitCode", LongType(), nullable=True),
                        StructField("JobExitCode", LongType(), nullable=True),
                        StructField("CMS_SubmissionTool", StringType(), nullable=True)
                    ]
                ),
            ),
        ]
    )
```


```python
_DEFAULT_HDFS_FOLDER = "/project/monitoring/archive/condor/raw/metric"
```


```python
def get_candidate_files(start_date, end_date, spark, base=_DEFAULT_HDFS_FOLDER):
    st_date = start_date - timedelta(days=3)
    ed_date = end_date + timedelta(days=3)
    days = (ed_date - st_date).days
    pre_candidate_files = [
        "{base}/{day}{{,.tmp}}".format(
            base=base, day=(st_date + timedelta(days=i)).strftime("%Y/%m/%d")
        )
        for i in range(0, days)
    ]
    sc = spark.sparkContext
    candidate_files = [
        f"{base}/{(st_date + timedelta(days=i)).strftime('%Y/%m/%d')}"
        for i in range(0, days)
    ]
    FileSystem = sc._gateway.jvm.org.apache.hadoop.fs.FileSystem
    URI = sc._gateway.jvm.java.net.URI
    Path = sc._gateway.jvm.org.apache.hadoop.fs.Path
    fs = FileSystem.get(URI("hdfs:///"), sc._jsc.hadoopConfiguration())
    candidate_files = [url for url in candidate_files if fs.globStatus(Path(url))]
    return candidate_files

```


```python
schema = _get_schema()
start_date = datetime(2022, 5, 1)
end_date = datetime(2022, 6, 1)
```


```python
get_candidate_files(start_date, end_date, spark, base=_DEFAULT_HDFS_FOLDER)
```




    ['/project/monitoring/archive/condor/raw/metric/2022/04/28',
     '/project/monitoring/archive/condor/raw/metric/2022/04/29',
     '/project/monitoring/archive/condor/raw/metric/2022/04/30',
     '/project/monitoring/archive/condor/raw/metric/2022/05/01',
     '/project/monitoring/archive/condor/raw/metric/2022/05/02',
     '/project/monitoring/archive/condor/raw/metric/2022/05/03',
     '/project/monitoring/archive/condor/raw/metric/2022/05/04',
     '/project/monitoring/archive/condor/raw/metric/2022/05/05',
     '/project/monitoring/archive/condor/raw/metric/2022/05/06',
     '/project/monitoring/archive/condor/raw/metric/2022/05/07',
     '/project/monitoring/archive/condor/raw/metric/2022/05/08',
     '/project/monitoring/archive/condor/raw/metric/2022/05/09',
     '/project/monitoring/archive/condor/raw/metric/2022/05/10',
     '/project/monitoring/archive/condor/raw/metric/2022/05/11',
     '/project/monitoring/archive/condor/raw/metric/2022/05/12',
     '/project/monitoring/archive/condor/raw/metric/2022/05/13',
     '/project/monitoring/archive/condor/raw/metric/2022/05/14',
     '/project/monitoring/archive/condor/raw/metric/2022/05/15',
     '/project/monitoring/archive/condor/raw/metric/2022/05/16',
     '/project/monitoring/archive/condor/raw/metric/2022/05/17',
     '/project/monitoring/archive/condor/raw/metric/2022/05/18',
     '/project/monitoring/archive/condor/raw/metric/2022/05/19',
     '/project/monitoring/archive/condor/raw/metric/2022/05/20',
     '/project/monitoring/archive/condor/raw/metric/2022/05/21',
     '/project/monitoring/archive/condor/raw/metric/2022/05/22',
     '/project/monitoring/archive/condor/raw/metric/2022/05/23',
     '/project/monitoring/archive/condor/raw/metric/2022/05/24',
     '/project/monitoring/archive/condor/raw/metric/2022/05/25',
     '/project/monitoring/archive/condor/raw/metric/2022/05/26',
     '/project/monitoring/archive/condor/raw/metric/2022/05/27',
     '/project/monitoring/archive/condor/raw/metric/2022/05/28',
     '/project/monitoring/archive/condor/raw/metric/2022/05/29',
     '/project/monitoring/archive/condor/raw/metric/2022/05/30',
     '/project/monitoring/archive/condor/raw/metric/2022/05/31',
     '/project/monitoring/archive/condor/raw/metric/2022/06/01',
     '/project/monitoring/archive/condor/raw/metric/2022/06/02',
     '/project/monitoring/archive/condor/raw/metric/2022/06/03']



### Get raw data from condor raw


```python
raw_df = (
        spark.read.option("basePath", _DEFAULT_HDFS_FOLDER)
        .json(
            get_candidate_files(start_date, end_date, spark, base=_DEFAULT_HDFS_FOLDER),
            schema=schema,
        ).select("data.*")
        .filter(
            f"""Status IN ('Completed') 
          AND RecordTime >= {start_date.timestamp() * 1000}
          AND RecordTime < {end_date.timestamp() * 1000}
          """
        )
        .drop_duplicates(["GlobalJobId"])
    )

spark.conf.set("spark.sql.session.timeZone", "UTC")
```


```python
raw_df.printSchema()
```

    root
     |-- CMSSite: string (nullable = true)
     |-- RecordTime: long (nullable = true)
     |-- InputData: string (nullable = true)
     |-- CMSPrimaryDataTier: string (nullable = true)
     |-- Status: string (nullable = true)
     |-- OverflowType: string (nullable = true)
     |-- WallClockHr: double (nullable = true)
     |-- CoreHr: double (nullable = true)
     |-- CpuTimeHr: double (nullable = true)
     |-- RequestCpus: long (nullable = true)
     |-- Type: string (nullable = true)
     |-- CRAB_DataBlock: string (nullable = true)
     |-- GlobalJobId: string (nullable = true)
     |-- ExitCode: long (nullable = true)
     |-- Chirp_CRAB3_Job_ExitCode: long (nullable = true)
     |-- Chirp_WMCore_cmsRun_ExitCode: long (nullable = true)
     |-- JobExitCode: long (nullable = true)
     |-- CMS_SubmissionTool: string (nullable = true)
    


### Task1

#### Sum of "WallClockHr" by "CMSPrimaryDataTier"



```python
df1 = raw_df.groupby([col("CMSPrimaryDataTier")])\
            .agg(_sum("WallClockHr").alias("Sum_WallClockHr"),\
                 _sum("CoreHr").alias("Sum_CoreHr"))\
            .sort("Sum_WallClockHr")
```


```python
df1_dict = _to_dict(df1)
```


```python
df1_wc = _other_fields(df1_dict['CMSPrimaryDataTier'], df1_dict['Sum_WallClockHr'], 2)
df1_core = _other_fields(df1_dict['CMSPrimaryDataTier'], df1_dict['Sum_CoreHr'], 2)
```


```python
dictlist1 = [{"index": df1_wc['index'],\
             "values": df1_wc['data_percent'],\
             "title": "Sum WallClockHr Per CMSPrimaryDataTier"},\
            {"index": df1_core['index'],\
             "values": df1_core['data_percent'],\
             "title": "Sum CoreHr Per CMSPrimaryDataTier"}
           ]
```


```python
_donut(dictlist1, "wc_core_datatier")
```


    
![png](/crab_data_analysis_doc/img/output_18_0.png)
    


### Task2

#### Sum of "WallClockHr" by "Type" ['production', 'analysis']



```python
df2 = raw_df.filter(col('Type').isin(['production', 'analysis']))\
            .groupby([col('Type')])\
            .agg(_sum("WallClockHr").alias("Sum_WallClockHr"),\
                 _sum("CoreHr").alias("Sum_CoreHr"))
```


```python
df2_dict = _to_dict(df2)
```


```python
dictlist2 = [{"index": df2_dict['Type'],\
             "values": df2_dict['Sum_WallClockHr'],\
             "title": "Sum WallClockHr Per Type"},\
            {"index": df2_dict['Type'],\
             "values": df2_dict['Sum_CoreHr'],\
             "title": "Sum CoreHr Per Type"}
           ]
```


```python
_pie(dictlist2, "wc_core_type")
```


    
![png](/crab_data_analysis_doc/img/output_24_0.png)
    


### Task3

#### Sum of "WallClockHr" filter "Type"['analysis'] by "CRAB_DataBlock" ['MCFakeBlock', Else]


```python
df3 = raw_df.filter(col('Type')=='analysis')\
            .withColumn('isMcprod', when(col('CRAB_DataBlock')=='MCFakeBlock', 'mc').otherwise('notMc'))\
            .groupby([col('isMcprod')])\
            .agg(_sum("WallClockHr").alias("Sum_WallClockHr"),\
                 _sum("CoreHr").alias("Sum_CoreHr"))
```


```python
df3_dict = _to_dict(df3)
```


```python
dictlist3 = [{"index": df3_dict['isMcprod'],\
             "values": df3_dict['Sum_WallClockHr'],\
             "title": "Sum WallClockHr Per isMcprod"},\
            {"index": df3_dict['isMcprod'],\
             "values": df3_dict['Sum_CoreHr'],\
             "title": "Sum CoreHr Per isMcprod"}
           ]
```


```python
_pie(dictlist3, "wc_core_ismcprod")
```


    
![png](/crab_data_analysis_doc/img/output_30_0.png)
    


### Task4

#### Average CPU Efficiency group by "RecordTime" each hour and "InputData" ['onsite', 'offsite']


```python
raw_df.createOrReplaceTempView("day")
df4 = spark.sql('SELECT     DATE_FORMAT(FROM_UNIXTIME(day.RecordTime/1000), "HH") AS timestamp,\
                            day.InputData AS inputData, \
                            SUM(day.CpuTimeHr) AS sumWallclockHr, \
                            SUM(day.WallClockHr*day.RequestCpus) as sumWallByReqCpus, \
                            SUM(day.CpuTimeHr)/SUM(day.WallClockHr*day.RequestCpus) AS avgCpuEff_Wall, \
                            SUM(day.CoreHr) as sumCoreHr, \
                            SUM(day.CpuTimeHr)/SUM(day.CoreHr) AS avgCpuEff_Core \
                    FROM day \
                    GROUP BY DATE_FORMAT(FROM_UNIXTIME(day.RecordTime/1000), "HH"), inputData\
                    ORDER BY DATE_FORMAT(FROM_UNIXTIME(day.RecordTime/1000), "HH")')
```


```python
df4_onsite = _to_dict(df4.filter(col('inputData')=='Onsite'))
df4_offsite = _to_dict(df4.filter(col('inputData')=='Offsite'))
```

    IOPub message rate exceeded.
    The notebook server will temporarily stop sending output
    to the client in order to avoid crashing it.
    To change this limit, set the config variable
    `--NotebookApp.iopub_msg_rate_limit`.
    
    Current values:
    NotebookApp.iopub_msg_rate_limit=1000.0 (msgs/sec)
    NotebookApp.rate_limit_window=3.0 (secs)
    



```python
figinfo  = {"x_label": "hour in a day","y_label": "avgCpuEff_Core","title": "AVG CPU Efficiency May 1st"}
```


```python
dictlist4 = [{"y-axis": [float(i) for i in df4_onsite['avgCpuEff_Wall']],\
             "label": "Onsite",\
             "color": "Orange"},\
            {"y-axis": [float(i) for i in df4_offsite['avgCpuEff_Wall']],\
             "label": "Offsite",\
             "color": "Blue"}
           ]
```


```python
hour = [float(i) for i in df4_onsite['timestamp']]
```


```python
_line_graph(hour, dictlist4, figinfo, "linegraph", False)
```


    
![png](/crab_data_analysis_doc/img/output_38_0.png)
    



```python
_line_graph(hour, dictlist4, figinfo, "linegraph", True)
```


    
![png](/crab_data_analysis_doc/img/output_39_0.png)
    


### Task5

#### Success rate of the "Type" ['analysis']


```python
df51 = raw_df.select(col('Type'), col('RecordTime'), col('JobExitCode'), col('Chirp_CRAB3_Job_ExitCode'), \
                     col('Chirp_WMCore_cmsRun_ExitCode'), col('ExitCode'))\
            .filter((col("Type")=="analysis")&(col("ExitCode").isNotNull()))\
            .sort('RecordTime')
```


```python
df52 = df51.select(col('ExitCode'))\
            .groupby(col('ExitCode'))\
            .agg(_count(col('ExitCode')).alias("count_ExitCode"))\
            .sort(col("count_ExitCode").desc())
```


```python
df52_dict = _to_dict(df52)
```


```python
df52_other = _other_fields(df52_dict['ExitCode'], df52_dict['count_ExitCode'], 2)
```


```python
dictlist52 = [{"index": df52_other['index'],\
             "values": df52_other['data_percent'],\
             "title": "count ExitCode"}
           ]
```


```python
_donut(dictlist52, "exitcode")
```


    
![png](/crab_data_analysis_doc/img/output_47_0.png)
    


### Additional Questions

1. Which are the most frequent reason for job failures in last week/month/year... as a function of time. which are the most relevant as wasted resources (i.e. weighted by used wall time on grid).


```python
df6 = raw_df.select(col('Type'), col('WallClockHr'), col('ExitCode'))\
            .filter((col("Type")=="analysis")&(col("ExitCode").isNotNull()))\
            .groupby(col('ExitCode'))\
            .agg(_count(col('ExitCode')).alias("count_ExitCode"),\
                 _sum("WallClockHr").alias("sum_WallClockHr"))\
            .sort('count_ExitCode')
```


```python
df6_dict = _to_dict(df6)
```


```python
df6_exitcode = _other_fields(df6_dict['ExitCode'], df6_dict['count_ExitCode'], 2)
df6_wallclock = _other_fields(df6_dict['ExitCode'], df6_dict['sum_WallClockHr'], 0.5)
```


```python
dictlist6 = [{"index": df6_exitcode['index'],\
             "values": df6_exitcode['data_percent'],\
             "title": "count ExitCode"},\
             {"index": df6_wallclock['index'],\
             "values": df6_wallclock['data_percent'],\
             "title": "sum wallclock"}
           ]
```


```python
_donut(dictlist6, "exitcode1")
print(_exitcode_info(0))
print(_exitcode_info(8028))
print(_exitcode_info(50666))
print(_exitcode_info(50660))
```


    
![png](/crab_data_analysis_doc/img/output_54_0.png)
    


    {'ExitCode': 0, 'Type': '', 'Meaning': ''}
    {'ExitCode': 8028, 'Type': 'cmsRun (CMSSW) exit codes. These codes may depend on specific CMSSW version', 'Meaning': 'FileOpenError with fallback'}
    {'ExitCode': 50666, 'Type': 'Failures related executable file', 'Meaning': ''}
    {'ExitCode': 50660, 'Type': 'Failures related executable file', 'Meaning': 'Application terminated by wrapper because using too much RAM (RSS)'}


### data from _DEFAULT_HDFS_FOLDER = "/project/monitoring/archive/condor/raw/metric" range 5/1 - 6/1


```python
schema = _get_schema()
start_date = datetime(2022, 5, 1)
end_date = datetime(2022, 6, 1)
```


```python
df7 = raw_df.select(col('WallClockHr'), col('ExitCode'))\
            .filter((col("CMS_SubmissionTool")=="CRAB")&(col("ExitCode").isNotNull()))\
            .groupby(col('ExitCode'))\
            .agg(_count(col('ExitCode')).alias("count_ExitCode"),\
                 _sum("WallClockHr").alias("sum_WallClockHr"))\
            .sort('count_ExitCode')
```


```python
df7_dict = _to_dict(df7)
```


```python
df7_exitcode = _other_fields(df7_dict['ExitCode'], df7_dict['count_ExitCode'], 0.5)
df7_wallclock = _other_fields(df7_dict['ExitCode'], df7_dict['sum_WallClockHr'], 0.5)
```


```python
dictlist7 = [{"index": df7_exitcode['index'],\
             "values": df7_exitcode['data_percent'],\
             "title": "count ExitCode"},\
             {"index": df7_wallclock['index'],\
             "values": df7_wallclock['data_percent'],\
             "title": "sum wallclock"}
           ]
```


```python
_donut(dictlist7, "checkwithgrafana")
```


    
![png](/crab_data_analysis_doc/img/output_61_0.png)
    


#### time range july 14


```python
start_date = datetime(2022, 7, 14)
end_date = datetime(2022, 7, 15)
```


```python
df8 = raw_df.select(col('WallClockHr'), col('ExitCode'))\
            .filter((col("CMS_SubmissionTool")=="CRAB")&(col("ExitCode").isNotNull()))\
            .groupby(col('ExitCode'))\
            .agg(_count(col('ExitCode')).alias("count_ExitCode"),\
                 _sum("WallClockHr").alias("sum_WallClockHr"))\
            .sort('sum_WallClockHr')
```


```python
df8_dict = _to_dict(df8)
```


```python
df8_exitcode = _other_fields(df8_dict['ExitCode'], df8_dict['count_ExitCode'], 0.5)
df8_wallclock = _other_fields(df8_dict['ExitCode'], df8_dict['sum_WallClockHr'], 0.5)
```


```python
dictlist8 = [{"index": df8_exitcode['index'],\
             "values": df8_exitcode['data_percent'],\
             "title": "count ExitCode"},\
             {"index": df8_wallclock['index'],\
             "values": df8_wallclock['data_percent'],\
             "title": "sum wallclock"}
           ]
```


```python
my_donut(dictlist8, "checkwithgrafana2")
```


    
![png](/crab_data_analysis_doc/img/output_68_0.png)
    



```python
print(_exitcode_info(8021))
print(_exitcode_info(60324))
```

    {'ExitCode': 8021, 'Type': 'cmsRun (CMSSW) exit codes. These codes may depend on specific CMSSW version', 'Meaning': 'FileReadError (May be a site error)'}
    {'ExitCode': 60324, 'Type': 'Failures related staging-OUT', 'Meaning': 'Other stageout exception.'}



```python

```
