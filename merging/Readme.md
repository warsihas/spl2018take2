[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **Merging the Data sets for final Data set** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml

Name of QuantLet : Merging data set


Description: Cleaning data and merging the original data sets.

Keywords: plot, vizualization, heatmap

Author: Gabriel Blumenstock, Felix Degenhardt, Haseeb Warsi


```



### R Code
```r
###load and filter datasets----
a <- read.csv("MCI_2014_to_2017.csv")
b <- read.csv("2016_neighbourhood_profiles.csv")
drugs <- read.csv("toronto_drug_arrests.csv")
wbt <- read.csv("wellbeing_toronto.csv")
area <- read_csv("toronto_area.csv")

colnames(a)[1] <- "X"

#Create aggregate data frame using data table
a.dt <- as.data.frame(a)

#Remove duplicated event IDS
a.dt <- subset(a.dt, !duplicated(a.dt$event_unique_id))

#Filter out occurrence dates before so we only look at 2016
a.dt[,c("occurrencedate", "reporteddate")] <- lapply(a.dt[,c("occurrencedate", "reporteddate")], as.Date)
a.dt <- a.dt %>%
           filter(occurrencedate >= as.Date("2016-01-01") & occurrencedate < as.Date("2017-01-01"))
a.dt <- a.dt[complete.cases(a.dt), ]

####Aggregate crimes by type----
a.dt <- as.data.table(a.dt)
setkey(a.dt, "MCI", "Hood_ID")
agg <- a.dt[, .(count = .N), by = c("MCI", "Hood_ID")]
agg <- dcast(agg, Hood_ID ~ MCI)
agg$Hood_ID <- as.factor(agg$Hood_ID)

#Turn Nas to 0 for crimes that didn't occur in that neighbourhood
agg[is.na(agg)] <- 0


###Add Drug Arrests to aggregate dataset----
drugs$Neighbourhood.Id <- as.factor(drugs$Neighbourhood.Id)
agg <- merge(agg, drugs[,c("Neighbourhood.Id", "Drug.Arrests")], by.x = "Hood_ID", by.y = "Neighbourhood.Id" )

###GEt crime totals for each neighbourhood
agg$Total.crime <- rowSums(agg[,-1])
colnames(agg) <- c("Hood_ID", "assault", "auto.theft", "break.and.enter", "robbery", "theft.over", "drug.arrests", "total.crime")

##Merge agg with wbt to get crime and neighbourhood profiles in 1 data frame
wbt$Neighbourhood.Id <- as.factor(wbt$Neighbourhood.Id)
agg.2014 <- merge(agg, wbt, by.x = "Hood_ID", by.y = "Neighbourhood.Id")
agg.2014[,c("Neighbourhood", "Combined.Indicators")] <- NULL

###Clean 2016 neighbourhood profile dataset to get variables---- 
###USing the 2016 neighbourhood profiles data to get 2016 data
df1 <- as.data.frame(b)
df1 <- b[-c(1,2),-c(1,3)]
str(df1)

###Create a dataframe of codes for each neighbourhood
neigh.codes <- as.data.frame(cbind(colnames(df1[,-c(1,2)]), as.vector(unlist(b[1,-c(1:4)]))))
neigh.codes <- neigh.codes[-1,] 
colnames(neigh.codes) <- c("Neighborhood", "Hood_ID")

#####Remove thousands seperator commas from numbers and replace % sign eith e-2, 
####so we can use as.numeric to convert from a character to a number
df1[,-c(1,2)] <- lapply(df1[,-c(1,2)], function(x) {gsub(",", "", x)})
df1[,-c(1,2)] <- lapply(df1[,-c(1,2)], function(x) {gsub("%", "e-2", x)})

###Turn n/as into NA
df1[df1 == "n/a"] <- NA

###Create function to get data from main dataset
getData <- function(x, characteristic) {
  a <- subset(x, Characteristic == characteristic) 
  a <-  as.data.frame(colSums(a[,-1]))
}

####Turn columns into numeric
df1[,-c(1,2)] <- lapply(df1[,-c(1,2)], as.numeric)
df1[,c("Topic", "City.of.Toronto")] <- NULL

###Create dataframe to aggregate all variables from 2016 neighbourhod profiles dataset
agg.2016 <- cbind.data.frame(Hood_ID = neigh.codes$Hood_ID)

###Age and Gender variables----
###Get number of males from 15 - 24 in 2016
df2 <- as.data.frame(df1)
df2 <- df2[c(14:55),]
df2$Characteristic <- fct_collapse(df2$Characteristic,
                         male.youth = c("Male: 15 to 19 years", "Male: 20 to 24 years")
                    )

male.youth <- cbind.data.frame(neigh.codes, getData(df2, "male.youth"))
colnames(male.youth) <- c(colnames(neigh.codes), "male.youth")
agg.2016 <- join(agg.2016, male.youth[, -1], by = "Hood_ID")

###Get number of youth (15 - 24) in 2016

df2 <- as.data.frame(df1)
df2 <- df2[c(14:55),]
df2$Characteristic <- fct_collapse(df2$Characteristic,
                                   youth = c("Male: 15 to 19 years", "Male: 20 to 24 years", "Female: 15 to 19 years", "Female: 20 to 24 years")
)

youth <- cbind.data.frame(neigh.codes, getData(df2, "youth"))
colnames(youth) <- c(colnames(neigh.codes), "youth")
agg.2016 <- join(agg.2016, youth[, -1], by = "Hood_ID")


####Get number of males in 2016
df2 <- as.data.frame(df1)
df2 <- df2[c(14:55),]
df2$Characteristic <- fct_collapse(df2$Characteristic,
                                   male.above.15 = c("Male: 15 to 19 years", "Male: 20 to 24 years", "Male: 25 to 29 years", 
                                                     "Male: 30 to 34 years", "Male: 35 to 39 years", "Male: 40 to 44 years", 
                                                     "Male: 45 to 49 years", "Male: 50 to 54 years", "Male: 55 to 59 years",
                                                     "Male: 60 to 64 years", "Male: 65 to 69 years", "Male: 70 to 74 years",
                                                     "Male: 75 to 79 years", "Male: 80 to 84 years", "Male: 85 to 89 years",
                                                     "Male: 90 to 94 years", "Male: 95 to 99 years", "Male: 100 years and over"),
                                   
                                   female.above.15 = c("Female: 15 to 19 years", "Female: 20 to 24 years", "Female: 25 to 29 years", 
                                                       "Female: 30 to 34 years", "Female: 35 to 39 years", "Female: 40 to 44 years", 
                                                       "Female: 45 to 49 years", "Female: 50 to 54 years", "Female: 55 to 59 years",
                                                       "Female: 60 to 64 years", "Female: 65 to 69 years", "Female: 70 to 74 years",
                                                       "Female: 75 to 79 years", "Female: 80 to 84 years", "Female: 85 to 89 years",
                                                       "Female: 90 to 94 years", "Female: 95 to 99 years", "Female: 100 years and over"))
#SUbset males above 15 years old
male.above.15 <- cbind.data.frame(neigh.codes, getData(df2, "male.above.15"))
colnames(male.above.15) <- c(colnames(neigh.codes), "male.above.15")
agg.2016 <- join(agg.2016, male.above.15[, -1], by = "Hood_ID")

#SUbset females above 15 years old
female.above.15 <- cbind.data.frame(neigh.codes, getData(df2, "female.above.15"))
colnames(female.above.15) <- c(colnames(neigh.codes), "female.above.15")
agg.2016 <- join(agg.2016, female.above.15[, -1], by = "Hood_ID")

#Get population for 2016
df2 <- as.data.frame(df1)

df2$Characteristic <- fct_collapse(df2$Characteristic,
                                   population.2016 = c("Population, 2016"))

population.2016 <- cbind.data.frame(neigh.codes, getData(df2, "population.2016"))
colnames(population.2016) <- c(colnames(neigh.codes), "population.2016")
agg.2016 <- join(agg.2016, population.2016[, -1], by = "Hood_ID")

###Get area and density of each neighbourhood
colnames(area)[c(2,3)] <- c("Hood_ID", "total.area")
agg.2016 <- join(agg.2016, area[,-1], by = "Hood_ID")
agg.2016$density <- agg.2016$population.2016 / agg.2016$total.area

###Get Lone Parent Families by sex of parent
df2 <- as.data.frame(df1)
df2 <- df2[c(88:94),]

###Total # of Lone parent families
lone.parent.families <- cbind.data.frame(neigh.codes, getData(df2, "Total lone-parent families by sex of parent"))
colnames(lone.parent.families) <- c(colnames(neigh.codes), "lone.parent.families")
agg.2016 <- join(agg.2016, lone.parent.families[, -1], by = "Hood_ID")

###percent of lone parent families
lone.parent.families.per <- cbind.data.frame(neigh.codes, (getData(df2, "Total lone-parent families by sex of parent") / 
                                getData(df2, "Total number of census families in private households") * 100))

colnames(lone.parent.families.per) <- c(colnames(neigh.codes), "lone.parent.families.per")
agg.2016 <- join(agg.2016, lone.parent.families.per[, -1], by = "Hood_ID")

###Get Income characteristices of each neighbourhood----
#####Get income groups
df2 <- as.data.frame(df1)

df2 <- df2[c(968:980),]
df2 <- df2[!df2$Characteristic == "$100,000 and over",]
df2$Characteristic <- fct_collapse(df2$Characteristic,
                                   low.income = c("Under $10,000 (including loss)", "$10,000 to $19,999",
                                                  "$20,000 to $29,999", "$30,000 to $39,999"),
                                   middle.income = c("$40,000 to $49,999", "$50,000 to $59,999", "$60,000 to $69,999",
                                                     "$70,000 to $79,999", "$80,000 to $89,999"),
                                   high.income = c("$90,000 to $99,999", "$100,000 to $149,999", "$150,000 and over"))

####GEt number of low income people in each neighbourhood
low.income <- cbind.data.frame(neigh.codes, getData(df2, "low.income"))
colnames(low.income) <- c(colnames(neigh.codes), "low.income")
agg.2016 <- join(agg.2016, low.income[, -1], by = "Hood_ID")

####GEt number of middle income people in each neighbourhood
middle.income <- cbind.data.frame(neigh.codes, getData(df2, "middle.income"))
colnames(middle.income) <- c(colnames(neigh.codes), "middle.income")
agg.2016 <- join(agg.2016, middle.income[, -1], by = "Hood_ID")

####GEt number of high income people in each neighbourhood
high.income <- cbind.data.frame(neigh.codes, getData(df2, "high.income"))
colnames(high.income) <- c(colnames(neigh.codes), "high.income")
agg.2016 <- join(agg.2016, high.income[, -1], by = "Hood_ID")

###Get average income
df2 <- as.data.frame(df1)
df2 <- df2[c(2261:2363),]

avg.income <- cbind.data.frame(neigh.codes, getData(df2, "Total income: Average amount ($)"))
colnames(avg.income) <- c(colnames(neigh.codes), "avg.income")
agg.2016 <- join(agg.2016, avg.income[, -1], by = "Hood_ID")

###number of people taking unemployment benefits (EI)
people.ei <- cbind.data.frame(neigh.codes, getData(df2, "Employment Insurance (EI) benefits: Population with an amount"))
colnames(people.ei) <- c(colnames(neigh.codes), "people.ei") 
agg.2016 <- join(agg.2016, people.ei[, -1], by = "Hood_ID")

###percent of people taking unemployment benefits (EI)
people.ei.per <- cbind.data.frame(neigh.codes, (getData(df2, "Employment Insurance (EI) benefits: Population with an amount") /
                    getData(df2, "Total income: Population with an amount") * 100))
colnames(people.ei.per) <- c(colnames(neigh.codes), "people.ei.per")
agg.2016 <- join(agg.2016, people.ei.per[, -1], by = "Hood_ID")

###Find Median Income
df2 <- as.data.frame(df1)
df2 <- df2[c(968:980),]
df2 <- df2[!df2$Characteristic == "$100,000 and over",]

###Change Characteristic Vector to specific form
df2$Characteristic <- c("0-9999", "10000-19999", "20000-29999", "30000-39999", "40000-49999", "50000-59999",
                        "60000-69999", "70000-79999", "80000-89999", "90000-99999", "100000-149999", "150000-1000000")

###Create Function to Calculate median income using groups
Grouped_Median <- function(frequencies, intervals, sep = NULL, trim = NULL) {
  # If "sep" is specified, the function will try to create the 
  #   required "intervals" matrix. "trim" removes any unwanted 
  #   characters before attempting to convert the ranges to numeric.
  if (!is.null(sep)) {
    if (is.null(trim)) pattern <- ""
    else if (trim == "cut") pattern <- "\\[|\\]|\\(|\\)"
    else pattern <- trim
    intervals <- sapply(strsplit(gsub(pattern, "", intervals), sep), as.numeric)
  }
  
  Midpoints <- rowMeans(intervals)
  cf <- cumsum(frequencies)
  Midrow <- findInterval(max(cf)/2, cf) + 1
  L <- intervals[1, Midrow]      # lower class boundary of median class
  h <- diff(intervals[, Midrow]) # size of median class
  f <- frequencies[Midrow]       # frequency of median class
  cf2 <- cf[Midrow - 1]          # cumulative frequency class before median class
  n_2 <- max(cf)/2               # total observations divided by 2
  
  unname(L + (n_2 - cf2)/f * h)
}

median.income <- cbind.data.frame(neigh.codes, 
                                  as.data.frame(sapply(df2[,-1], function(x) {Grouped_Median(x,intervals = df2$Characteristic, sep = "-")})))
colnames(median.income) <- c(colnames(neigh.codes), "median.income")
agg.2016 <- join(agg.2016, median.income[, -1], by = "Hood_ID")

#####Calculate number of Households in bottom 20% of Income distribution
df2 <- as.data.frame(df1)
df2 <- df2[c(1106:1116),]
df2 <- df2[!df2$Characteristic == "In the top half of the distribution",]

##Calculate sum of households in bottom 20% of income distribution
no.hholds.bottom.20per <- cbind.data.frame(neigh.codes, 
                                           as.data.frame(colSums(df2[df2$Characteristic == "In the bottom decile" | df2$Characteristic == "In the second decile", -1])))
colnames(no.hholds.bottom.20per) <- c(colnames(neigh.codes), "no.hholds.bottom.20per") 
agg.2016 <- join(agg.2016, no.hholds.bottom.20per[, -1], by = "Hood_ID")

##Calculate percentage of households in bottom 20% of income distribution
hholds.bottom.20per.per <- cbind.data.frame(neigh.codes, 
                                            round(no.hholds.bottom.20per[, "no.hholds.bottom.20per"] / colSums(df2[,-1]), 2) * 100)
colnames(hholds.bottom.20per.per) <- c(colnames(neigh.codes), "hholds.bottom.20per.per")
agg.2016 <- join(agg.2016, hholds.bottom.20per.per[, -1], by = "Hood_ID")

###Get number of low income individuals
df2 <- as.data.frame(df1)
df2 <- df2[c(1122:1131),]

#Total number of low income individuals according to low income measure
low.income.pop <- cbind.data.frame(neigh.codes, 
                                   getData(df2, "In low income based on the Low-income measure, after tax (LIM-AT)"))
colnames(low.income.pop) <- c(colnames(neigh.codes), "low.income.pop")
agg.2016 <- join(agg.2016, low.income.pop[, -1], by = "Hood_ID")

###Number of low income individuals according to low income measure 18-64 years
low.income.pop.18.to.64 <- cbind.data.frame(neigh.codes, getData(df2, "18 to 64 years"))
colnames(low.income.pop.18.to.64) <- c(colnames(neigh.codes), "low.income.pop.18.to.64")  
agg.2016 <- join(agg.2016, low.income.pop.18.to.64[, -1], by = "Hood_ID")

###Percentage of low income individuals according to low income measure
low.income.pop.per <- cbind.data.frame(neigh.codes, 
                                       getData(df2, "Prevalence of low income based on the Low-income measure, after tax (LIM-AT) (%)"))
colnames(low.income.pop.per) <- c(colnames(neigh.codes), "low.income.pop.per")  
agg.2016 <- join(agg.2016, low.income.pop.per[, -1], by = "Hood_ID")

###Percenatage of low income individuals according to low income measure 18-64 years
low.income.pop.18.to.64.per <- cbind.data.frame(neigh.codes, getData(df2, "18 to 64 years (%)"))
colnames(low.income.pop.18.to.64.per) <- c(colnames(neigh.codes), "low.income.pop.18.to.64.per")  
agg.2016 <- join(agg.2016, low.income.pop.18.to.64.per[, -1], by = "Hood_ID")

###Citizenship and Immigration stats of residents----
###GEt number of non-Canadian citizens in each neighbourhood
df2 <- as.data.frame(df1)
df2 <- df2[c(1142:1146),]

####Get number of non-Canadian citizens in each neighborhood
non.citizens <- cbind.data.frame(neigh.codes, getData(df2, "Not Canadian citizens"))
colnames(non.citizens) <- c(colnames(neigh.codes), "non.citizens")
agg.2016 <- join(agg.2016, non.citizens[, -1], by = "Hood_ID")

###Get percentage of non-citizens in neighbourhood
non.citizens.per <- cbind.data.frame(neigh.codes, 
                                     getData(df2, "Not Canadian citizens") / getData(df2, "Total - Citizenship for the population in private households - 25% sample data") * 100)
colnames(non.citizens.per) <- c(colnames(neigh.codes), "non.citizens.per")
agg.2016 <- join(agg.2016, non.citizens.per[, -1], by = "Hood_ID")

###Get number of immigrants in each neighbourhood
df2 <- as.data.frame(df1)
df2 <- df2[c(1147:1156),]

###Get number of immigrants in each neighbourhood
immigrants <- cbind.data.frame(neigh.codes, getData(df2, "Immigrants"))
colnames(immigrants) <- c(colnames(neigh.codes), "immigrants")
agg.2016 <- join(agg.2016, immigrants[, -1], by = "Hood_ID")

###Get percentage of immigrants in each neighbourhood
immigrants.per <- cbind.data.frame(neigh.codes,
                                   getData(df2, "Immigrants") / getData(df2, "Total - Immigrant status and period of immigration for the population in private households - 25% sample data") * 100)
colnames(immigrants.per) <- c(colnames(neigh.codes), "immigrants.per")
agg.2016 <- join(agg.2016, immigrants.per[, -1], by = "Hood_ID")

####Number of immigrants in last 5 years as recent immigrants
immigrants.recent <- cbind.data.frame(neigh.codes, getData(df2, "2011 to 2016"))
colnames(immigrants.recent) <- c(colnames(neigh.codes), "immigrants.recent")
agg.2016 <- join(agg.2016, immigrants.recent[, -1], by = "Hood_ID")

###percentage of population that are recent immigrants
immigrants.recent.per <- cbind.data.frame(neigh.codes, 
                                          getData(df2, "2011 to 2016") / getData(df2, "Total - Immigrant status and period of immigration for the population in private households - 25% sample data") * 100)
colnames(immigrants.recent.per) <- c(colnames(neigh.codes), "immigrants.recent.per")
agg.2016 <- join(agg.2016, immigrants.recent.per[, -1], by = "Hood_ID")

####Get number of refugees who landed from 1986-2016 in each neighbourhood
df2 <- as.data.frame(df1)
df2 <- df2[c(1289:1295),]

####Get number of refugees who landed from 1986-2016 in each neighbourhood
refugees <- cbind.data.frame(neigh.codes, getData(df2, "Refugees"))
colnames(refugees) <- c(colnames(neigh.codes), "refugees")
agg.2016 <- join(agg.2016, refugees[, -1], by = "Hood_ID")

####Get percent of refugees who landed from 1986-2016 in each neighbourhood
refugees.per <- cbind.data.frame(neigh.codes,
                                 getData(df2, "Refugees") / population.2016[, "population.2016"] * 100) 
colnames(refugees.per) <- c(colnames(neigh.codes), "refugees.per")
agg.2016 <- join(agg.2016, refugees.per[, -1], by = "Hood_ID")

####Get number of visible minorities in each neighbourhood
df2 <- as.data.frame(df1)
df2 <- df2[c(1330:1344),]

####Get number of visible minorities in each neighbourhood
vis.minorities <- cbind.data.frame(neigh.codes, getData(df2, "Total visible minority population"))
colnames(vis.minorities) <- c(colnames(neigh.codes), "vis.minorities")
agg.2016 <- join(agg.2016, vis.minorities[ , -1], by = "Hood_ID")

###Get percent of pop that are visible minorities
vis.minorities.per <- cbind.data.frame(neigh.codes, 
                                       getData(df2, "Total visible minority population") / getData(df2, "Total - Visible minority for the population in private households - 25% sample data") * 100)
colnames(vis.minorities.per) <- c(colnames(neigh.codes), "vis.minorities.per")
agg.2016 <- join(agg.2016, vis.minorities.per[, -1], by = "Hood_ID")

###Housing characteristics----
###Get number of renters
df2 <- as.data.frame(df1)
df2 <- df2[c(1624:1626),]

renters <- cbind.data.frame(neigh.codes, getData(df2, "Renter"))
colnames(renters) <- c(colnames(neigh.codes), "renters")
agg.2016 <- join(agg.2016, renters[, -1], by = "Hood_ID")

####Get percent of renters in each neighbourhood
renters.per <- cbind.data.frame(neigh.codes, 
                                getData(df2, "Renter") / getData(df2, "Total - Private households by tenure - 25% sample data") * 100)
colnames(renters.per) <- c(colnames(neigh.codes), "renters.per")
agg.2016 <- join(agg.2016, renters.per[, -1], by = "Hood_ID")

###Get number of dwellings that are not condominiums in each neighbourhood
df2 <- as.data.frame(df1)
df2 <- df2[c(1628:1630),]

houses <- cbind.data.frame(neigh.codes, getData(df2, "Not condominium"))
colnames(houses) <- c(colnames(neigh.codes), "houses")
agg.2016 <- join(agg.2016, houses[, -1], by = "Hood_ID")

###Get percent of buildings that are not condominiums in each neighbourhood
houses.per <- cbind.data.frame(neigh.codes, 
                               getData(df2, "Not condominium") / getData(df2, "Total - Occupied private dwellings by condominium status - 25% sample data") * 100) 
colnames(houses.per) <- c(colnames(neigh.codes), "houses.per")
agg.2016 <- join(agg.2016, houses.per[, -1], by = "Hood_ID")

###Get number of households living in unsuitable housing conditions for size and makeup of family, 
###according to statscanada definition
df2 <- as.data.frame(df1)
df2 <- df2[c(1646:1648),]

unsuitable.housing <- cbind.data.frame(neigh.codes, getData(df2, "Not suitable"))
colnames(unsuitable.housing) <- c(colnames(neigh.codes), "unsuitable.housing")
agg.2016 <- join(agg.2016, unsuitable.housing[, -1], by = "Hood_ID")

###percent hholds in unsuitable housing
unsuitable.housing.per <- cbind.data.frame(neigh.codes,
                                           getData(df2, "Not suitable") / getData(df2, "Total - Private households by housing suitability - 25% sample data") * 100) 
colnames(unsuitable.housing.per) <- c(colnames(neigh.codes), "unsuitable.housing.per")
agg.2016 <- join(agg.2016, unsuitable.housing.per[, -1], by = "Hood_ID")

###Get households that require major repairs
df2 <- as.data.frame(df1)
df2 <- df2[c(1657:1659),]

hhlds.mjr.rprs <- cbind.data.frame(neigh.codes, getData(df2, "Major repairs needed"))
colnames(hhlds.mjr.rprs) <- c(colnames(neigh.codes), "hhlds.mjr.rprs")
agg.2016 <- join(agg.2016, hhlds.mjr.rprs[, -1], by = "Hood_ID")

###Get percent or hhlds that need major repairs
hhlds.mjr.rprs.per <- cbind.data.frame(neigh.codes, 
                                       getData(df2, "Major repairs needed") / getData(df2, "Total - Occupied private dwellings by dwelling condition - 25% sample data") * 100)
colnames(hhlds.mjr.rprs.per) <- c(colnames(neigh.codes), "hhlds.mjr.rprs.per")
agg.2016 <- join(agg.2016, hhlds.mjr.rprs.per[, -1], by = "Hood_ID")

###Get households that spend 30 percent or more of income on shelter costs
df2 <- as.data.frame(df1)
df2 <- df2[c(1673:1676),]

more.than.30per.on.shltr <- cbind.data.frame(neigh.codes, getData(df2, "Spending 30% or more of income on shelter costs"))
colnames(more.than.30per.on.shltr) <- c(colnames(neigh.codes), "unaffordable.housing")
agg.2016 <- join(agg.2016, more.than.30per.on.shltr[, -1], by = "Hood_ID")

###Get percent of households that spend more than 30 percent of income on shelter costs
more.than.30per.on.shltr.per <- cbind.data.frame(neigh.codes, getData(df2, "Spending 30% or more of income on shelter costs") /
                                    getData(df2, "Total - Owner and tenant households with household total income greater than zero; in non-farm; non-reserve private dwellings by shelter-cost-to-income ratio - 25% sample data") * 100)

colnames(more.than.30per.on.shltr.per) <- c(colnames(neigh.codes), "unaffordable.housing.per")
agg.2016 <- join(agg.2016, more.than.30per.on.shltr.per[, -1], by = "Hood_ID")

###Education of residents-----
###Get data on education level of residents of each neighbourhood

df2 <- as.data.frame(df1)
df2 <- df2[c(1698:1701),]

###number of people with less than high school certificate

less.than.high.school <- cbind.data.frame(neigh.codes, getData(df2, "No certificate, diploma or degree"))  
colnames(less.than.high.school) <- c(colnames(neigh.codes), "less.than.high.school") 
agg.2016 <- join(agg.2016, less.than.high.school[, -1], by = "Hood_ID")

###percent of people with less than high school certificate

less.than.high.school.per <- cbind.data.frame(neigh.codes, 
                                              getData(df2, "No certificate, diploma or degree") / getData(df2, "Total - Highest certificate, diploma or degree for the population aged 15 years and over in private households - 25% sample data") * 100)
colnames(less.than.high.school.per) <- c(colnames(neigh.codes), "less.than.high.school.per")
agg.2016 <- join(agg.2016, less.than.high.school.per[, -1], by = "Hood_ID")

###number of people with high school or equivalent certificate

high.school.cert <- cbind.data.frame(neigh.codes, getData(df2, "Secondary (high) school diploma or equivalency certificate"))  
colnames(high.school.cert) <- c(colnames(neigh.codes), "high.school.cert")
agg.2016 <- join(agg.2016, high.school.cert[, -1], by = "Hood_ID")

###percent of people with less than high school certificate

high.school.cert.per <- cbind.data.frame(neigh.codes,
                                         getData(df2, "Secondary (high) school diploma or equivalency certificate") / getData(df2, "Total - Highest certificate, diploma or degree for the population aged 15 years and over in private households - 25% sample data") * 100) 
colnames(high.school.cert.per) <- c(colnames(neigh.codes), "high.school.cert.per")
agg.2016 <- join(agg.2016, high.school.cert.per[, -1], by = "Hood_ID")

###number of people with post-secondary education or higher
post.sec.or.above <- cbind.data.frame(neigh.codes, getData(df2, "Postsecondary certificate, diploma or degree"))
colnames(post.sec.or.above) <- c(colnames(neigh.codes), "post.sec.or.above")
agg.2016 <- join(agg.2016, post.sec.or.above[, -1], by = "Hood_ID")

###percent of people with post-secondary education or higher
post.sec.or.above.per <- cbind.data.frame(neigh.codes, 
                                          getData(df2, "Postsecondary certificate, diploma or degree") / getData(df2, "Total - Highest certificate, diploma or degree for the population aged 15 years and over in private households - 25% sample data") * 100) 
colnames(post.sec.or.above.per) <- c(colnames(neigh.codes), "post.sec.or.above.per")
agg.2016 <- join(agg.2016, post.sec.or.above.per[, -1], by = "Hood_ID")

###Employment Data-----
###Get data on unemployment level of each neighbourhood

df2 <- as.data.frame(df1)
df2 <- df2[c(1880:1895),]

###Get number of unemployed people in each neighbourhood
unemployed <- cbind.data.frame(neigh.codes, getData(df2, "Unemployed"))
colnames(unemployed) <- c(colnames(neigh.codes), "unemployed")
agg.2016 <- join(agg.2016, unemployed[, -1], by = "Hood_ID")

###Get unemployment rate for each neighbourhood
unemployment.rate <- cbind.data.frame(neigh.codes, getData(df2, "Unemployment rate"))
colnames(unemployment.rate) <- c(colnames(neigh.codes), "unemployment.rate")
agg.2016 <- join(agg.2016, unemployment.rate[, -1], by = "Hood_ID")

###Get number of unemployed males
unemployed.males <- cbind.data.frame(neigh.codes, getData(df2, "Unemployed (Males)"))
colnames(unemployed.males) <- c(colnames(neigh.codes), "unemployed.males")
agg.2016 <- join(agg.2016, unemployed.males[, -1], by = "Hood_ID")

###Get unemployment rate for males
unemployment.rate.males <- cbind.data.frame(neigh.codes, getData(df2, "Unemployment rate (Males)"))
colnames(unemployment.rate.males) <- c(colnames(neigh.codes), "unemployment.rate.males")
agg.2016 <- join(agg.2016, unemployment.rate.males[, -1], by = "Hood_ID")

###Merge crime data with 2016 agg data
agg.2016 <- as.data.frame(join(agg, agg.2016, by = "Hood_ID"))

