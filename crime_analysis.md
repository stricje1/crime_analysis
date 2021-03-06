---
title: "Crime Analysis using R"
author: "Jeffrey Strickland"
date: "2/22/2022"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE, message=FALSE, warning=FALSE, fig.width=6.5, fig.height=4, scipen = 1000000)
```

## Introduction

# Objectives
  - Visualizing spatial and temporal trends of criminal activity
  - Analyzing factors that may affect unlawful behavior

## Install R libraries

  ```{r crime01, echo = TRUE}
if(!require(readr)) install.packages("readr")
if(!require(dplyr)) install.packages("dplyr")
if(!require(DT)) install.packages("DT")
if(!require(ggrepel)) install.packages("ggrepel")
if(!require(leaflet)) install.packages("leaflet")
```
# the Crime Data
## About the Data

The data we use for this chapter was simulated. I will now explain how this was accomplished. First, we examined crime data from similar sized cities and developed their probability distributions. For instance Larcent/theft was approximatley uniformly distributed Uniform~[120000,160000] with mean 140000 and standard deviation of approximately 11547. To simulate larceny/theft in Atlanta, we used the reverse inverse method, which for the Uniform distribution is X = a + (b - a)U, where U is the Uniform[0,1] random number. In Excel, this is the RAND() function, so the inverse transform would be X = a + (b - a) *RAND(). The crimes are then geographically distributed in zones in a similar fashion. Due to the hish resolution of the San Fransico crime data, we used their lat/longs as a basline and converted them to atlanta as follows (using center-of-city). San Francisco is loated at (37.733795	-122.446747) and Atlant is at (33.753746,-84.38633), yolding a difference (3.980049,-38.060417). Atlanta police zones wehre alos assigned randomly.

The resulting set is atlanta_crime_4yr.xls, with 515807 records and thirteen variables

- IncidntNum	(T)	Incident number
- Category	(T)	Crime category, i.e., larceny/theft
- Descript 	(T)
- DayOfWeek 	(T)
- Date 	(D	Date: DD/MM/YYYY
- Time 	(T)	Time: 24-hour system
- PdDistrict	(T)	Police district where incident occured
- Resolution	(T)	Resolution of the crime
- Address 	(T)	Address of the crime
- X 	(N)	Longitude
- Y 	(N)	Latitude
- Location 	(T) Lat/long
- PdId 	(N)	Police Department ID


## Read the data
### Load the data using readr and read_csv().

```{r crime02, echo = TRUE}
library(readr)
# path <- "https://github.com/stricje1/crime_analysis/blob/main/atlanta_crime_10yr.zip/atlanta_crime_10yr.zip"
path <- "c:\\Users\\jeff\\Documents\\VIT_University\\data\\atlanta_crime_4yr.csv"
df <- read.csv(path)
```

## Display Data
### Display the data using DT and datatable().


library(DT)
df_sub <- df[1:100,]  # display the first 100 rows
df_sub$Time <- as.character(df_sub$Time) 
datatable(df_sub, options = list(pageLength = 5,scrollX='400px'))

Here we apply sprintf, a wrapper for the C function sprintf, that returns a character vector containing a formatted combination of text and variable values.

```{r crime04, echo = TRUE}
sprintf("Number of Rows in Dataframe: %s", format(nrow(df),big.mark = ","))
```

## Preprocess Data
# The All-Caps text is difficult to read. Let???s force the text in the appropriate columns into proper case.

```{r crime05, echo = TRUE}
str(df)
proper_case <- function(x) {
  return (gsub("\\b([A-Z])([A-Z]+)", "\\U\\1\\L\\2" , x, perl=TRUE))
}
```


library(dplyr)
df <- df %>% mutate(Category = proper_case(Category),
                    Descript = proper_case(Descript),
                    PdDistrict = proper_case(PdDistrict),
                    Resolution = proper_case(Resolution),
                    Time = as.character(Time))
df_sub <- df[1:100,]  # display the first 100 rows
datatable(df_sub, options = list(pageLength = 5,scrollX='400px'))

# Visualize Data

## Crime across space
In this section, we use the leaflet function. It creates a Leaflet map widget using htmlwidgets. The widget can be rendered on HTML pages generated from R Markdown. In addition to matrices and data frames, leaflet supports spatial objects from the sp package and spatial data frames from the sf package.
We create a Leaflet map with these basic steps: First, create a map widget by calling leaflet(). Next, we add layers (i.e., features) to the map by using layer functions (e.g. addTiles, addMarkers, addPolygons) to modify the map widget. Then you keep adding layers or stop when satified with the result.
We will add a tile layer from a known map provider, using the leaflet function addProviderTiles. A list of providers can be found at http://leaflet-extras.github.io/leaflet-providers/preview/.  We will also add graphics elements and layers to the map widget with addCondtroll(addTiles).
We use markers to call out points on the map. Marker locations are expressed in latitude/longitude coordinates, and can either appear as icons or as circles. When there are a large number of markers on a map as in our case with crimes, we can cluster them together.
Now, we define crime incident locations on the map using leaflet, which we use for our popups. 

```{r crime07, echo = TRUE}
library(leaflet)
data <- df[1:10000,] # display the first 10,000 rows
data$popup <- paste("<b>Incident #: </b>", data$IncidntNum, "<br>", "<b>Category: </b>", data$Category,
                    "<br>", "<b>Description: </b>", data$Descript,
                    "<br>", "<b>Day of week: </b>", data$DayOfWeek,
                    "<br>", "<b>Date: </b>", data$Date,
                    "<br>", "<b>Time: </b>", data$Time,
                    "<br>", "<b>PD district: </b>", data$PdDistrict,
                    "<br>", "<b>Resolution: </b>", data$Resolution,
                    "<br>", "<b>Address: </b>", data$Address,
                    "<br>", "<b>Longitude: </b>", data$X,
                    "<br>", "<b>Latitude: </b>", data$Y)
```
In this manner, we can click icons on the map to show incident details. We need to set up some generate some parameters that we concatenate or "paste" together to form these incident descriptions. For example, the concatenated strings pdata$popup, provides the content of the second incident as shown here:

```{r crime08, echo = TRUE}
data$popup[1]
```

You may notice the "%>%" or forward-pipe operator in the leaflet arguments. The operators pipe their left-hand side values forward into expressions that appear on the right-hand side, rather than from the inside and out.

```{r crime09, echo = TRUE}
leaflet(data, width = "100%") %>% addTiles() %>%
  addTiles(group = "OSM (default)") %>%
  addProviderTiles(provider = "Esri.WorldStreetMap",group = "World StreetMap") %>%
  addProviderTiles(provider = "Esri.WorldImagery",group = "World Imagery") %>%
  # addProviderTiles(provider = "NASAGIBS.ViirsEarthAtNight2012",group = "Nighttime Imagery") %>%
  addMarkers(lng = ~X, lat = ~Y, popup = data$popup, clusterOptions = markerClusterOptions()) %>%
  addLayersControl(
    baseGroups = c("OSM (default)","World StreetMap", "World Imagery"),
    options = layersControlOptions(collapsed = FALSE)
  )
```
## Crime Over Time
That was not meant to rhyme, but I like it. 
In this section, we will manipulate the data using the dplyr::mutate function. mutate adds new variables while preserving extisting variables. Below, we used "shades of bue" in the code for our plot, with a dark blue line that smooths the data. 

```{r crime10, echo = TRUE}
library(dplyr)

df_crime_daily <- df %>%
  mutate(Date = as.Date(Date, "%m/%d/%Y")) %>%
  group_by(Date) %>%
  summarize(count = n()) %>%
  arrange(Date)
```
## Daily Crimes Plot with Variance
```{r crime11, echo = TRUE}
library(ggplot2)
library(scales)
plot <- ggplot(df_crime_daily, aes(x = Date, y = count)) +
  geom_line(color = "#50a8ff", size = 0.1) +
  geom_smooth(color = "#00008b") +
  # fte_theme() +
  scale_x_date(breaks = date_breaks("1 year"), labels = date_format("%Y")) +
  labs(x = "Date of Crime", y = "Number of Crimes", title = "Daily Crimes in Atlanta from 2014 ??? 2018")
plot
```

## Aggregate Data
No, wcan aggregate the data and create a table that summarizes the data by incident category. (In Rstudio, this table shows in the Viewer pane.)
We used the descending order of "decreasing" for sorting the incident category. DT::datatable or the datatable function generates the HTML table widget.

```{r crime12, echo = TRUE}
df_category <- sort(table(df$Category),decreasing = TRUE)
df_category <- data.frame(df_category[df_category > 7000])
colnames(df_category) <- c("Category", "Frequency")
df_category$Percentage <- df_category$Frequency / sum(df_category$Frequency)
```
datatable(df_category, options = list(scrollX='400px'))

## Create a Bar Chart
Now that we can aggregate the data, we will show the data with a bar graph:
  
```{r crime13, echo = TRUE}
library(ggplot2)
library(ggrepel)
bp<-ggplot(df_category, aes(x=Category, y=Frequency, fill=Category)) + geom_bar(stat="identity") + 
  theme(axis.text.x=element_blank()) + geom_text_repel(data=df_category, aes(label=Category))
bp
```

## Create a pie chart based on the incident category.
To further illustrate the crime incident data, a subsequent pie chart is plotted.

```{r crime14, echo = TRUE}
bp<-ggplot(df_category, aes(x="", y=Percentage, fill=Category)) + geom_bar(stat="identity") 
pie <- bp + coord_polar("y") 
pie
```
# Temporal Trends

## Theft Over Time

In this section, we create a chart of crimes (Larceny/Theft) over time. And for aesthetic effect, we make using shades of green (Shades of Green is an Armed Forces Recreation Center (AFRC) resort at Walt Disney World).

```{r crime15, echo = TRUE}
df_theft <- df %>% filter(grepl("Larceny/Theft", Category))

df_theft_daily <- df_theft %>%
  mutate(Date = as.Date(Date, "%m/%d/%Y")) %>%
  group_by(Date) %>%
  summarize(count = n()) %>%
  arrange(Date)

library(ggplot2)
library(scales)
plot <- ggplot(df_theft_daily, aes(x = Date, y = count)) +
  geom_line(color = "#00cc00", size = 0.1) +
  geom_smooth(color = "#008000") +
  # fte_theme() +
  scale_x_date(breaks = date_breaks("1 year"), labels = date_format("%Y")) +
  labs(x = "Date of Theft", y = "Number of Thefts", title = "Daily Thefts in Atlanta from 2014 ??? 2018")
plot
```

## Theft Time Heatmap
Now, we aggregate counts of thefts by Day-of-Week and Time to create heat map. Fortunately, the Day-Of-Week part is pre-derived, but Hour is slightly harder.
We need a function that gets the hour from the time string in atlanta_crime_4yr.csv, so that we can use an approximate arrest time with day of the week. But R does not have one, or one I can find. So, we build the function below, using the colon delimiter.

```{r crime16, echo = TRUE}
get_hour <- function(x) {
  return (as.numeric(strsplit(x,":")[[1]][1]))
}

df_theft_time <- df_theft %>%
  mutate(Hour = sapply(Time, get_hour)) %>%
  group_by(DayOfWeek, Hour) %>%
  summarize(count = n())
# df_theft_time %>% head(10)

datatable(df_theft_time, options = list(scrollX='400px'))
```

## Reorder and format Factors
In this section, we demonstrate how ro reorder and format factors using the aggregated data. For instance, the rev function reverses elements so that the days of the week are "Saturday", "Friday", "Thursday", "Wednesday", "Tuesday", "Monday" and "Sunday." We use the factor function to encode a vector of times as a factor (the terms ???category??? and ???enumerated type??? are also used for factors). 

```{r crime16a, echo = TRUE}
dow_format <- c("Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday")
hour_format <- c(paste(c(12,1:11),"AM"), paste(c(12,1:11),"PM"))

df_theft_time$DayOfWeek <- factor(df_theft_time$DayOfWeek, level = rev(dow_format))
df_theft_time$Hour <- factor(df_theft_time$Hour, level = 0:23, label = hour_format)

# df_theft_time %>% head(10)
datatable(df_theft_time, options = list(scrollX='400px'))
```

## Create Time Heatmap
Using our previus results, we build a "heatmap" and plot it with a red color scheme.If you have not noticed, most of the colors I am using are in hexadecimal (hex) numbers, like "#000000" instead of black. A great website to get colors with hex number is https://www.colorhexa.com/000000.

```{r crime18, echo = TRUE}
plot <- ggplot(df_theft_time, aes(x = Hour, y = DayOfWeek, fill = count)) +
  geom_tile() +
  # fte_theme() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.6), legend.title = element_blank(), legend.position="top", legend.direction="horizontal", legend.key.width=unit(2, "cm"), legend.key.height=unit(0.25, "cm"), legend.margin=unit(-0.5,"cm"), panel.margin=element_blank()) +
  labs(x = "Hour of Theft (Local Time)", y = "Day of Week of Theft", title = "Number of Thefts in Atlanta from 2014 ??? 2018, by Time of Theft") +
  scale_fill_gradient(low = "white", high = "#d40000", labels = comma)
plot
```
The graph brings up a question: why is there a surge at 6-7PM on weekdays?

## Arrest Over Time
Now, we create a chart of arrests over time. First, we setup the data to get arrest counts by date. Then we plot the number of thefts given the date of the theft.
  
```{r crime19, echo = TRUE}
df_arrest <- df %>% filter(grepl("Arrest", Resolution))

df_arrest_daily <- df_arrest %>%
  mutate(Date = as.Date(Date, "%m/%d/%Y")) %>%
  group_by(Date) %>%
  summarize(count = n()) %>%
  arrange(Date)
```

## Daily Arrests
Next, we build a plot of Daily Ploice Arrests  

```{r crime20, echo = TRUE}
library(ggplot2)
library(scales)
plot <- ggplot(df_arrest_daily, aes(x = Date, y = count)) +
  geom_line(color = "#cd00cd", size = 0.1) +
  geom_smooth(color = "#1A1A1A") +
  # fte_theme() +
  scale_x_date(breaks = date_breaks("1 year"), labels = date_format("%Y")) +
  labs(x = "Date of Arrest", y = "# of Police Arrests", title = "Daily Police Arrests in Atlanta from 2014 ??? 2018")
plot
```
## Number of Arrest by TIme of Arrest

Here, we again use the function we created that gets the hour from the time string in atlanta_crime_4yr.csv, so that we can use an approximate arrest time with day of the week. This allos us to bin the crimes by hour, etc.

```{r crime21, echo = TRUE}
df_arrest_time <- df_arrest %>%
  mutate(Hour = sapply(Time, get_hour)) %>%
  group_by(DayOfWeek, Hour) %>%
  summarize(count = n())

dow_format <- c("Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday")
hour_format <- c(paste(c(12,1:11),"AM"), paste(c(12,1:11),"PM"))

df_arrest_time$DayOfWeek <- factor(df_arrest_time$DayOfWeek, level = rev(dow_format))
df_arrest_time$Hour <- factor(df_arrest_time$Hour, level = 0:23, label = hour_format)

plot <- ggplot(df_arrest_time, aes(x = Hour, y = DayOfWeek, fill = count)) +
  geom_tile() +
  # fte_theme() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.6), legend.title = element_blank(), legend.position="top", legend.direction="horizontal", legend.key.width=unit(2, "cm"), legend.key.height=unit(0.25, "cm"), legend.margin=unit(-0.5,"cm"), panel.margin=element_blank()) +
  labs(x = "Hour of Arrest (Local Time)", y = "Day of Week of Arrest", title = "Number of Police Arrests in Atlanta from 2014 ??? 2018, by Time of Arrest") +
  scale_fill_gradient(low = "white", high = "#008000", labels = comma)
plot
```
Why is there a surge on Wednesday afternoon, and at 4-5PM on all days? Let???s look at subgroups to verify there isn???t a latent factor.

# Correlation Analysis

## Factor by Crime Category

Certain types of crime may be more time dependent. (e.g., more traffic violations when people leave work)

```{r crime22, echo = TRUE}
library(dplyr)
df_top_crimes <- df_arrest %>%
  group_by(Category) %>% 
  summarize(count = n()) %>%
  arrange(desc(count))

datatable(df_top_crimes, options = list(pageLength = 10,scrollX='400px'))

df_arrest_time_crime <- df_arrest %>%
  filter(Category %in% df_top_crimes$Category[2:19]) %>%
  mutate(Hour = sapply(Time, get_hour)) %>%
  group_by(Category, DayOfWeek, Hour) %>% 
  summarize(count = n())

df_arrest_time_crime$DayOfWeek <- factor(df_arrest_time_crime$DayOfWeek, level = rev(dow_format))
df_arrest_time_crime$Hour <- factor(df_arrest_time_crime$Hour, level = 0:23, label = hour_format)

datatable(df_arrest_time_crime, options = list(pageLength = 10,scrollX='400px'))
```

## Number of Arrests by Category and time of Arrest

```{r crime23, echo = TRUE}
library(ggplot2)
plot <- ggplot(df_arrest_time_crime, aes(x = Hour, y = DayOfWeek, fill = count)) +
  geom_tile() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.6, size = 4)) +
  labs(x = "Hour of Arrest (Local Time)", y = "Day of Week of Arrest", title = "Number of Police Arrests in Atlanta from 2014 ??? 2018, by Category and Time of Arrest") +
  scale_fill_gradient(low = "white", high = "#2980B9") +
  facet_wrap(~ Category, nrow = 6)
plot
```

Good, but the gradients aren???t helpful because they are not normalized. We need to normalize the range on each facet. (unfortunately, this makes the value of the gradient unhelpful)

## Normailzed Gradients


df_arrest_time_crime <- df_arrest_time_crime %>%
  group_by(Category) %>%
  mutate(norm = count/sum(count))

datatable(df_arrest_time_crime, options = list(pageLength = 10,scrollX='400px'))

## Normalized Number of Arrests by Category and Time of Arrest

```{r crime25, echo = TRUE}
plot <- ggplot(df_arrest_time_crime, aes(x = Hour, y = DayOfWeek, fill = norm)) +
  geom_tile() +
  # fte_theme() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.6, size = 4)) +
  labs(x = "Hour of Arrest (Local Time)", y = "Day of Week of Arrest", title = "Police Arrests in Atlanta from 2014 ??? 2018 by Time of Arrest, Normalized by Type of Crime") +
  scale_fill_gradient(low = "white", high = "#2980B9") +
  facet_wrap(~ Category, nrow = 6)
plot
```

## Factor by Police District

We plot the same ame as above, but with a different facet.

df_arrest_time_district <- df_arrest %>%
  mutate(Hour = sapply(Time, get_hour)) %>%
  group_by(PdDistrict, DayOfWeek, Hour) %>% 
  summarize(count = n()) %>%
  group_by(PdDistrict) %>%
  mutate(norm = count/sum(count))

df_arrest_time_district$DayOfWeek <- factor(df_arrest_time_district$DayOfWeek, level = rev(dow_format))
df_arrest_time_district$Hour <- factor(df_arrest_time_district$Hour, level = 0:23, label = hour_format)

datatable(df_arrest_time_district, options = list(pageLength = 10,scrollX='400px'))

## Factor by Police District

```{r crime26, echo = TRUE}
plot <- ggplot(df_arrest_time_district, aes(x = Hour, y = DayOfWeek, fill = norm)) +
  geom_tile() +
  # fte_theme() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.6, size = 4)) +
  labs(x = "Hour of Arrest (Local Time)", y = "Day of Week of Arrest", title = "Police Arrests in Atlanta from 2014 ??? 2018 by Time of Arrest, Normalized by Station") +
  scale_fill_gradient(low = "white", high = "#8E44AD") +
  facet_wrap(~ PdDistrict, nrow = 5)
plot
```

## Factor by Month

We now look at factor by month. If crime is tied to activities, the period at which activies end may impact.

```{r crime27, echo = TRUE}
df_arrest_time_month <- df_arrest %>%
  mutate(Month = format(as.Date(Date, "%m/%d/%Y"), "%B"), Hour = sapply(Time, get_hour)) %>%
  group_by(Month, DayOfWeek, Hour) %>% 
  summarize(count = n()) %>%
  group_by(Month) %>%
  mutate(norm = count/sum(count))
```

# Here, we set order of month facets by chronological order instead of alphabetical.
```{r crime28, echo = TRUE}
df_arrest_time_month$DayOfWeek <- factor(df_arrest_time_month$DayOfWeek, level = rev(dow_format))
df_arrest_time_month$Hour <- factor(df_arrest_time_month$Hour, level = 0:23, label = hour_format)
df_arrest_time_month$Month <- factor(df_arrest_time_month$Month,
                                     level = c("January","February","March","April","May","June","July","August","September","October","November","December"))
# Plot of Factor by Month
  
  
```{r crime29, echo = TRUE}
plot <- ggplot(df_arrest_time_month, aes(x = Hour, y = DayOfWeek, fill = norm)) +
  geom_tile() +
  # fte_theme() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.6, size = 4)) +
  labs(x = "Hour of Arrest (Local Time)", y = "Day of Week of Arrest", title = "Police Arrests in Atlanta from 2007 ??? 2016 by Time of Arrest, Normalized by Month") +
  scale_fill_gradient(low = "white", high = "#E74C3C") +
  facet_wrap(~ Month, nrow = 4)
plot
```
## Factor By Year

#what if things changed overtime?

```{r crime30, echo = TRUE}
  df_arrest_time_year <- df_arrest %>%
  mutate(Year = format(as.Date(Date, "%m/%d/%Y"), "%Y"), Hour = sapply(Time, get_hour)) %>%
  group_by(Year, DayOfWeek, Hour) %>% 
  summarize(count = n()) %>%
  group_by(Year) %>%
  mutate(norm = count/sum(count))

df_arrest_time_year$DayOfWeek <- factor(df_arrest_time_year$DayOfWeek, level = rev(dow_format))
df_arrest_time_year$Hour <- factor(df_arrest_time_year$Hour, level = 0:23, label = hour_format)
```

## Police Arrest Normalized by YEar

```{r crime31, echo = TRUE}
plot <- ggplot(df_arrest_time_year, aes(x = Hour, y = DayOfWeek, fill = norm)) +
  geom_tile() +
  # fte_theme() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.6, size = 4)) +
  labs(x = "Hour of Arrest (Local Time)", y = "Day of Week of Arrest", title = "Police Arrests in Atlanta from 2014 ??? 2018 by Time of Arrest, Normalized by Year") +
  scale_fill_gradient(low = "white", high = "#E67E22") +
  facet_wrap(~ Year, nrow = 6)
plot
````

## Works CIted

- Data Provided By City and County of San Francisco. Source Link http://www.sfgov.org. License: Public Domain Dedication and License

