# Star Sharks ETL Report
Rebecca Blackham<br>
Valor Israel<br>
Jesse Noss

<br>

# Data Sources
The data we used to supplement this investigation was extracted from various sources. 

As our primary dataset, we used a [How Much Fish do People Eat?](https://ourworldindata.org/fish-and-overfishing#how-much-fish-do-people-eat)<sup>1</sup> dataset from [OurWorldInData](https://ourworldindata.org/), displaying the per capita seafood consumption in different countries. From this same source, we also pulled a few other datasets: a [Total Seafood Production by Country](https://ourworldindata.org/fish-and-overfishing#total-seafood-production-by-country) table, which provided us with a list of countries and their individual contributions to global seafood production; a [Protein from Seafood Proportion of Total Protein](https://ourworldindata.org/fish-and-overfishing#how-much-of-our-protein-comes-from-seafood) table, and a [Protein from Seafood Proportion of All Animal Protein](https://ourworldindata.org/fish-and-overfishing#how-much-of-our-animal-protein-comes-from-seafood) table, which look at the average number of kilograms each person in a nation takes in from different sources.  

As another metric, we added in a [Historical Population](https://ourworldindata.org/grapher/population)<sup>2</sup>, supplying us with population figures of countries around the world, with recorded data for more recent years and historical estimates for years further in the past. This data was pulled from OurWorldInData, which compiled a set sourced from, the [Hyde Database](https://www.pbl.nl/en/image/links/hyde)<sup>3</sup>, [Gapminder](https://www.gapminder.org/about/)<sup>4</sup>, and the [United Nations](https://population.un.org/wpp/Download/Standard/Population/)<sup>5</sup>. 

To address life expectancy, we brought in a [Global Life Expectancy Historical Dataset](https://www.kaggle.com/datasets/hasibalmuzdadid/global-life-expectancy-historical-dataset)<sup>6</sup> from [Kaggle](https://www.kaggle.com/).

Finally, we added continent labels by country, using a [Country & Area Dataset](https://www.kaggle.com/datasets/lukexun/country-area-continent-region-codes)<sup>7</sup>, also from Kaggle. 

<br>

# Extraction and Transformation
Our steps in processing the data were initially conducted through [VSCode python notebooks](./Workspaces/Vals_workspace/ETL.ipynb), where we cleaned up the datasets and ensured that they worked as we expected. We then continued the process in Azure Databricks, using the cleaned and joined datasets for further steps to produce and consume Kafka messages and deliver our data into an SQL database for querying and storage.

<br>

## Reading in CSVs
To begin this process, we gathered together the necessary csv files which were previously downloaded from the sources outlined in the [Data Sources](##-Data-Sources) section and given more clear/concise file names. These csv files were then read into their own pandas dataframes in python, from which point we could start cleaning them up and joining them together. The titles of these dataframes are outlined below, for easier reference throughout this section:

`consumption = pd.read_csv('../../Data/Fish/fish-and-seafood-consumption-per-capita.csv')`<br>
`production = pd.read_csv('../../Data/Fish/fish-seafood-production.csv')`<br>
`population = pd.read_csv('../../Data/Fish/population.csv')`<br>
`life_expect = pd.read_csv('../../Data/to_merge/life_expectancy.csv')`<br>
`c_n_c = pd.read_csv('../../Data/to_merge/country_and_continents.csv')`<br>
`protein_all = pd.read_csv('../../Data/all_protein_consumption_sources.csv')`<br>
`protein_animal = pd.read_csv('../../Data/animal_protein_consumption.csv')`

<br>

## Initial Dataframe Transformations
For each of the newly created dataframes, we made sure that columns were named appropriately, simplifying those which had been labeled with complex titles: 

- In the `production` dataframe, we renamed the existing column "`Fish and seafood | 00002960 || Production | 005511 || tonnes`" to "`Production_in_tons`"
- In the `consumption` dataframe, we renamed the existing column "`Fish and seafood | 00002960 || Food available for consumption | 0645pc || kilograms per year per capita`" to "`kg_per_year_per_capita`"
- In the `c_n_c` dataframe, we renamed the existing column "`country_or_area`" to "`Entity`"
- In the `protein_all` dataframe, we renamed the existing column "`percent_fish_total_protein_per_capita`" to "`% fish protein from all sources`"
- In the `protein_animal` dataframe, we renamed the existing column "`percent_fish_protein_all_animal`" to "`% fish protein from animal sources`"

<br>
Next, we made sure that the years the datasets covered matched up properly. After looking through each of them, we'd found that only one the Population dataframe had to be altered. We filtered the years of this dataframe by adding a clause `population['Year'] > 1960`, which limited its results to only those afer 1960. 

<br>
Finally, we had to restructure our `life_expect` dataframe, as it was set up with "Years" as columns, alongside "Country" and "Country Code", and the various values as rows, whereas our other dataframes listed all the years in a single column, attributed to rows for each of their respective countries. To do so, we used Python's Pandas `melt` function, "unpivoting" the dataframe to give us columns of `Country Name`, `Country Code`, `Year`, and `Value` (the life expectancy per country per year). To finish transforming this dataframe, we had to reassign specific datatypes to a couple columns, and rename the `Value` column:

- `Year` column changed to `int` data type
- `Value` column changed to `float` data type
- `Value` column renamed to "`Life_Expectancy_years`"

<br>

## Merging Dataframes
Following the initial transformations, we then merged our dataframes together to create a single dataframe that we would use for later processing, machine learning, and visualizations.

To do so, we used the `consumption` dataframe as a base, joining on columns from the rest of the dataframes on the following parameters:

- `['Year', 'Code', 'Production_in_tons']` columns from the `production` dataframe, merged on matching `Year` and `Code`
- `['Code', 'Year', 'Population (historical estimates)', 'Entity']` columns from the `population` dataframe, merged on matching `Code`, `Year`, and `Entity`
- The entire modified `life_expect` dataframe, merged on `'Entity'=='Country Name'`, `'Code'=='Country Code'`, and matching `Year`
- At this point, redundant columns of `['Country Name', 'Country Code']` were dropped from the merged dataframe
- The `c_n_c` dataframe was merged on matching `Entity`

As further additions, the `% fish protein from all sources` and the kg of protein columns from various sources from the `protein_all` dataframe, along with the `% fish protein from animal sources` column from the `protein_animal` dataframe, were merged together on matching `['Entity', 'Code', 'Year']` columns. These were used separately from the main dataframe for the creation of later visualizations. 

<br>

## Exporting to CSV
Once the final dataframe was put together from the merges, we exported it out as a CSV file with the Pandas function `.to_csv()`.

<br>

## Stepping into Databricks
Within a shared Databricks workspace, we set up a pipeline which would be used to move our new, cleaned, CSV file into a storage container in Azure and into a SQL database. 

To do so, we uploaded the CSV into an initial storage container. We then set up a mount point to access this storage container directly from the Databricks workspace, and used this connection to read the contents of the CSV first into a Spark dataframe, and then converting that into a Pandas dataframe for convenience - the conversion step isn't entirely necessary, but we were all more comfortable using Pandas than Spark dataframes. 

<br>

## Kafka Pipeline
With the mount point established and the data read in, we set up a Kafka Topic, Producer, and Consumer.

Using the Kafka Producer, we sought to read the lines of our dataframe as individual messages, sending each one at a specified interval to be registered under our created Topic, and timestamped appropriately. We then used the Kafka Consumer to read back those messages we'd sent, recording them into a list and, once all messages were retrieved, recreating a dataframe with all the necessary contents.

This final dataframe would then be saved into our blob storage, and uploaded into our SQL database,

<br>

## SQL
As a final step, we performed some data normalization on the file uploaded into SQL, splitting the dataframe up into multiple tables, and tying them together with defined relationships, as displayed below.

![](./Images/ERD.png)

<br>
<br>

# References
[finish adding citations to datasets]

<sup>1</sup>Ritchie, Hannah, and Max Roser. “Fish and Overfishing.” Our World in Data, October 2021. https://ourworldindata.org/fish-and-overfishing#. 

<sup>2</sup>“Population.” Our World in Data, March 31, 2023. https://ourworldindata.org/grapher/population. 

<sup>3</sup>Klein Goldewijk, K., A. Beusen, J.Doelman and E. Stehfest (2017), Anthropogenic land use estimates for the Holocene; HYDE 3.2, Earth System Science Data, 9, 927-953.

<sup>4</sup>Gapminder population dataset version 7, based on data by Angus Maddison improved by Clio Infra. https://www.gapminder.org/data/documentation/gd003/.

<sup>5</sup>United Nations, Department of Economic and Social Affairs, Population Division (2022). World Population Prospects 2022, Online Edition. Rev. 1.

<sup>6</sup>Muzdadid, Hasib Al. “Global Life Expectancy Historical Dataset.” Kaggle, November 26, 2022. https://www.kaggle.com/datasets/hasibalmuzdadid/global-life-expectancy-historical-dataset. 

<sup>7</sup>X, Luke. “Country &amp; Area Dataset.” Kaggle, May 23, 2022. https://www.kaggle.com/datasets/lukexun/country-area-continent-region-codes. 