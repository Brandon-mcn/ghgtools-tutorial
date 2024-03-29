ghgtools-tutorial
================
Brandon-mcn
March 3, 2024

Welcome to the ghgtools inventory data module!

Add your activity data template to the GHG Inventory Report project
folder. You can either drag and drop activity data into the provided
template ‘ghgtools_ActivityData’ or overwrite this file with your own
activity data file. Please ensure the data is formatted correctly and
saved as ‘ghgtools_ActivityData.xlsx’

Add your asset portfolio data template to the GHG Inventory Report
project folder. You can either drag and drop activity data into the
provided template ‘ghgtools_AssetPortfolio_V1’ or overwrite this file
with your own activity data file. Please ensure the data is formatted
correctly and saved as ‘ghgtools_AssetPortfolio_V1.xlsx’

``` r
#initiate ghgtools

if (!require("data.table")) install.packages("data.table")
```

    ## Loading required package: data.table

``` r
library(data.table)

#Select GWP 

GWP <- "AR5"

#load emission factor library

EFL <- fread("EFL.csv")

#load in asset portfolio 

AssetPortfolio <- fread("AssetPortfolio.csv")

#load in activity data

ActivityData <- fread("ActivityData.csv")

#load in eGRID lookup table for electricity subregions
  
eGRIDlookup <- fread("eGRID_lookup.csv")

#load in emission_category lookup
  
Ecat_lookup <- fread("Ecat_lookup.csv")
  
#load in GWPs and filter to selected assessment report
  
GWPs <- fread("GWPs.csv")
GWPs <- GWPs[, .(ghg,get(GWP))]
colnames(GWPs)[2] <- "GWP"

#Create EFL_CO2e table

EFL_CO2e <- data.table(merge.data.table(EFL, GWPs, sort = FALSE, all.x = TRUE))
EFL_CO2e[, sum_co2e := ghg_emission_factor * GWP]
EFL_CO2e <- EFL_CO2e[, .(kgco2e_perunit = sum(sum_co2e)), by = .(ef_source, ef_publishdate, ef_activeyear, service_type, unit, emission_category, service_subcategory1, service_subcategory2, emission_scope, country, subregion)]
EFL_CO2e[, ef_publishdate := format(ef_publishdate, "%m/%d/%Y")]
EFL_CO2e[EFL_CO2e == ""] <- NA
setnames(EFL_CO2e, "ef_activeyear", "year")

#ghgtools Initiate complete. Now user needs to add their data
#------------------------------------------------------------

DT1 <- fread("ActivityData.csv")
DT1[DT1 == ""] <- NA
DT2 <- data.table(merge.data.table(DT1, AssetPortfolio, sort = FALSE, all.x = TRUE))
DT3 <- data.table(merge.data.table(DT2, Ecat_lookup, by = c("asset_type", "service_type"), sort = FALSE))
DT4 <- data.table(merge.data.table(DT3, eGRIDlookup, sort = FALSE, all.x = TRUE))
GHGrawdata <- data.table(merge.data.table(DT4, EFL_CO2e, by = c("year", "service_type", "emission_category", "service_subcategory1", "service_subcategory2", "country", "subregion", "unit"), all.x = TRUE, sort = FALSE))
GHGrawdata[, kg_co2e := usage * kgco2e_perunit]
GHGrawdata[, MT_co2e := kg_co2e/1000]
setcolorder(GHGrawdata, c("asset_id", "asset_type", "asset_subtype", "address", "city", "state", "zip", "country", "region", "subregion", "business_unit", "year_built", "sqft", "service_type", "unit", "vendor", "account_id", "meter_number", "date", "year", "cost", "usage", "emission_category", "service_subcategory1", "service_subcategory2", "emission_scope", "kgco2e_perunit", "kg_co2e", "MT_co2e", "ef_source", "ef_publishdate"))
fwrite(GHGrawdata, "GHGrawdata.csv")
```

``` r
ghg_summary <- GHGrawdata[, .(EmissionTotal = sum(MT_co2e)), by = emission_scope]

library(ggplot2)
GHG_sum_chart <- ggplot(ghg_summary, aes(x = emission_scope, y = EmissionTotal)) +
  geom_bar(stat = "identity", fill = "#4B5320", color = "black") +
  labs(title = "GHG Emissions Total",
       x = "",
       y = "MT CO2e")
print(GHG_sum_chart)
```

![](GHG-Inventory-Data_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->
