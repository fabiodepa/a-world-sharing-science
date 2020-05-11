
# In a world sharing science

*autho*r: Fabio De Pascale

*date*: 8/5/2020

## Methodology note

This document describe the sources of data and the manipulation steps for an article I published on Medium [In a world sharing science](https://medium.com/@depascale.f/in-a-world-sharing-science-d15be7e92319).

### Data sources

Data were collected on the 30th of April 2020.

* From the github repository of [Nextstrain](https://github.com/nextstrain/ncov/) I downloaded this [table](https://github.com/nextstrain/ncov/data/metadata.tsv) that contain the metadata of the genomes submitted to Gisaid initiative.

* From [WHO](https://covid19.who.int/) I downloaded the data map of the number of confirmed cases of covid-19 around the world.

### Hand curation of WHO cases dataset:

This dataset starts from the 11 of January 2020 with the first 41 cases reported in China. Initially I thought to produce an evolution through time of the cases around the world thus all countries must cover the same time series

```{sh eval=FALSE, echo=TRUE}
# in shell
grep China WHO-COVID-19-global-data.csv | cut -d ',' -f1 > days.txt
```

Collapse countries in a single records with time series on columns.

```{sh eval=FALSE, echo=TRUE}
# with python3.6
## build date indexes
i=0
g=open("days.txt", 'r') # days from china rows
days={}
for line in g:
  line=line.rstrip('\n')
  days[line]=i
  i+=1

## read dataset and fill dictionary so to have one record per country
f=open("WHO-COVID-19-global-data.csv", 'r')
next(f)
newtab={}
for line in f:
  line=line.rstrip('\n')
  line=line.split(',')
  if line[2] in newtab:
    newtab[line[2]][2][days[line[0]]]=int(line[7])
  else:
    newtab[line[2]]=[line[1],line[3],[0]*111]
    newtab[line[2]][2][days[line[0]]]=int(line[7])
f.close()

## print results
o=open('WHO_transposed.tsv', 'w')
my_lst_str = '\t'.join(days)
header="Name" + "\t" + "Country" + "\t" + "Region" + "\t" + my_lst_str + '\n'
o.write(header)
for count in newtab:
  my_lst=newtab[count][2]
  my_lst_str = '\t'.join(map(str, my_lst))
  line=count + "\t" + newtab[count][0] + "\t" +  newtab[count][1] + "\t" + my_lst_str + '\n'
  o.write(line)
o.close()
```

In Flourish it was not possible to plot the point chart with a time series thus also the number of confirmed cases remain static at the 30 of April.

### Hand curation of GISAID dataset

GISAID metadata dataset contains the submitting lab that is the laboratory that submitted the sequences and thus it is useful to show scientific research efforts.

I manually corrected the redundant names and removed all those records that were submitted with a genome shorter than 29 Kbp. The reference is a bit longer (29.9 Kbp) but we can assume that records classified as partial being longer than 29 Kbp could be classifed as complete for the purpose of this article. Two records with submission dates 10 January 2020 were changed to the 11 of January in order to be consistent with WHO dataset that start from 11 January.

```{sh eval=FALSE, echo=TRUE}
# with ipython3.6
## retrive time series
i=0
g=open("days.txt", 'r') # days from china rows
days={}
for line in g:
  line=line.rstrip('\n')
  days[line]=i
  i+=1

## read dataset and fill dictionary one record for institute
f=open("genome_sequenced_sarscov2_gisaid_selected.txt", 'r')
next(f)
submittin_labs={}
for line in f:
     line=line.rstrip('\n')
     line=line.split('\t')
     if line[8] in submittin_labs:
         submittin_labs[line[8]][4][days[line[10]]]=submittin_labs[line[8]][3]+1
         submittin_labs[line[8]][3]+=1
     else:
         submittin_labs[line[8]]=[line[4],line[5],line[6],1,[0]*111]
         submittin_labs[line[8]][4][days[line[10]]]=1
f.close()

# replace 0 in rows so to have the commulative sum on the rows
test = submittin_labs
for inst in test:
     value = 0
     my_list=test[inst][4]
     new_list=[]
     for item in my_list: 
         if item != 0: 
             new_list.append(item) 
             value=item 
         else: 
             new_list.append(value) 
     test[inst][4]=new_list 

## print output
o=open('gisaid_institues_transposed.tsv', 'w')
my_lst_str = '\t'.join(days)
header="Institute" + "\t" + "Region" + "\t" + "Country" + "\t" + "Division" + "\t" + my_lst_str + '\n'
o.write(header)
for inst in submittin_labs:
  my_lst=submittin_labs[inst][4]
  my_lst_str = '\t'.join(map(str, my_lst))
  line=inst + "\t" + submittin_labs[inst][0] + "\t" + submittin_labs[inst][1] + "\t" + submittin_labs[inst][2] + "\t" + my_lst_str + '\n'
  o.write(line)
  
o.close()
```

Then I retrived the coordinates for the submitting institute home division, more or less the province where the institute is. 

```{sh eval=FALSE, echo=TRUE}
# in shell
ct -f4 gisaid_institues_transposed.tsv > division_gisaid_inst.txt

# in ipython3
import geopy
from geopy import geocoders
from geopy.geocoders import Nominatim
nom=Nominatim()
f=open("division_gisaid_inst.txt", 'r')
locations={}
for line in f:
    line=line.rstrip('\n')
    locations[line]=nom.geocode(line) 
f.close()
newdict={}
for item in locations:
    try:
      lat=locations[item].latitude
      lon=locations[item].longitude
    except:
      lat='Nan'
      lon='Nan' 
    newdict[item]=[lat,lon] 

o=open("division_gisaid_inst_coor.txt", 'w')
line= "Division\tLAT\tLON\n"
o.write(line)
for item in newdict:
  line=item + "\t" + str(newdict[item][0]) + "\t" + str(newdict[item][1]) + "\n"
  o.write(line) 
o.close()

# in shell
while read line; do 
  LAT=$( grep -w "^$line" division_gisaid_inst_coor.txt | cut -f2 ); 
  LON=$( grep -w "^$line" division_gisaid_inst_coor.txt | cut -f3 ); 
  echo -e "${line}\t${LAT}\t${LON}";
done <division_gisaid_inst.txt | tr '.' ',' > coordinates_division.txt
```

Change coordinates so to have point for decimal position.

With this dataset and the one from WHO I created the map with [Flourish](https://flourish.studio/)

The next step is useful to produce the second chart. I needed the commulative number of submission to GISAID for each continent.

```{sh eval=FALSE, echo=TRUE}
# in ipython3
import pandas as pd
df=pd.read_csv("gisaid_institues_transposed.tsv", sep="\t")

## group for region - continent - field and apply sum
final=df.groupby(['Region', 'Country']).sum()

## print output
final.to_csv("region_country_gisaid_submission", sep='\t')
```

And these data were plotted in the [Datawrapper](https://www.datawrapper.de/) area chart.

