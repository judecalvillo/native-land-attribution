![](img/header_native-land-los-angeles.png)

Native American Land Attribution for L.A. County
------------------------------------------------

### Overview

This **\#dataengineering** project, commissioned by [Hack for
LA](https://www.hackforla.org/) leadership, entails developing a
reusable pipeline for attributing L.A. County cities/parcels to the
Native American tribes whose homes they originally were. In doing so, I
leverage public resources and semi-unstructured data (in PDFs) to
deliver an adaptive **\#dataproduct** that can be used for multiple
purposes.

The steps in this pipeline are as follows, and the code (in **\#R**) is
below…

1.  Extract/parse city -&gt; zip code [lookup tables from LA County’s
    official
    PDF](http://file.lacounty.gov/SDSInter/lac/1031552_MasterZipCodes.pdf)  
2.  Clean up, normalize, and merge extracted lookup tables  
3.  Use [Google’s Geocoding
    API](https://developers.google.com/maps/documentation/geocoding/overview)
    to get the lat/lons per Zip  
4.  Query [Native Land Digital’s API](https://native-land.ca/) to find
    the Native American peoples (and languages) who originally lived
    there  
5.  Deliver the data product (currently in CSV format; can easily push
    to some DB)

For questions, please feel free to contact: [Jude
Calvillo](https://www.linkedin.com/in/judecalvillo). Thanks!

------------------------------------------------------------------------

    ## Load libraries
    library(tabulizer) # for extracting tables from PDFs
    library(jsonlite) # for neatly converting JSON to dataframes
    library(stringr) # for string splitting
    library(dplyr) # for data transformations/aggregations
    library(ggplot2) # for plotting
    library(httr) # for raw API calls
    library(ggmap) # easy interface to Google Geocode API

### Step 1. Parse Zip Code lookup table from LA County’s official PDF

Using Tabulizer library to extract tables from a PDF, then previewing
the data.

    ## Parse the Zip code lookup table from LA County's PDF at: http://file.lacounty.gov/SDSInter/lac/1031552_MasterZipCodes.pdf
    zips <- extract_tables("http://file.lacounty.gov/SDSInter/lac/1031552_MasterZipCodes.pdf")

    ## Preview
    head(zips[[1]])

    ##      [,1]                                        [,2]                          
    ## [1,] ""                                          "County of Los Angeles"       
    ## [2,] ""                                          "ZIP CODE LIST"               
    ## [3,] ""                                          ""                            
    ## [4,] "ZIP CODE"                                  "AREA NAME * (See note below)"
    ## [5,] "90001 Florence/South Central (City of LA)" ""                            
    ## [6,] "90002 Watts (City of LA)"                  ""                            
    ##      [,3]                    
    ## [1,] ""                      
    ## [2,] ""                      
    ## [3,] "Supervisorial District"
    ## [4,] "1st 2nd 3rd 4th 5th"   
    ## [5,] "X X"                   
    ## [6,] "X"

    head(zips[[2]])

    ##      [,1]    [,2]                                                  [,3] [,4]
    ## [1,] "90047" "South Central (City of LA)"                          ""   "X" 
    ## [2,] "90048" "West Beverly (City of LA)"                           ""   "X" 
    ## [3,] "90049" "Bel Air Estates (City of LA)/Brentwood (City of LA)" ""   ""  
    ## [4,] "90050" "Los Angeles"                                         "X"  ""  
    ## [5,] "90051" "Los Angeles"                                         ""   "X" 
    ## [6,] "90052" "Los Angeles"                                         ""   "X" 
    ##      [,5] [,6] [,7]
    ## [1,] ""   ""   ""  
    ## [2,] "X"  ""   ""  
    ## [3,] "X"  ""   ""  
    ## [4,] ""   ""   ""  
    ## [5,] ""   ""   ""  
    ## [6,] ""   ""   ""

    head(zips[[7]])

    ##      [,1]                             [,2] [,3]
    ## [1,] "91301 Agoura/Oak Park"          "X"  ""  
    ## [2,] "91302 Calabasas/Hidden Hills"   "X"  ""  
    ## [3,] "91303 Canoga Park (City of LA)" "X"  ""  
    ## [4,] "91304 Canoga Park (City of LA)" "X"  "X" 
    ## [5,] "91305 Canoga Park (City of LA)" "X"  ""  
    ## [6,] "91306 Winnetka (City of LA)"    "X"  ""

### Step 2. Clean up, normalize, and merge zip tables

Header and column detection were a little problematic on a few pages,
but the errors fall within a limited range of classes and the data is
still usable. Therefore, let’s just trim to what we need. This involves…

1.  Grabbing only the first two columns per table (given the commonality
    between the errors)  
2.  Remove all rows that don’t start with a zip code  
3.  For all those rows with zip and area name merged, split the text and
    replace  
4.  Flag areas that are officially part of the City of LA

–

    ## Generally, between the different types of errors we're seeing, the data we need is in the first 2 columns of the listed matrices, so let's row bind the first two columns of all the matrices
    zips_df <- data.frame()
    for(i in 1:length(zips)){
        zip_page <- as.data.frame(zips[i])[,c(1:2)]
        zips_df <- rbind(zips_df, zip_page)
    }

    ## Pretty the names and remove rows that don't start with 5 numbers, then preview
    names(zips_df) <- c("zip","area_name")
    zips_df <- zips_df[grep("^[0-9][0-9][0-9][0-9][0-9].*", as.character(zips_df$zip)),]
    head(zips_df)

    ##                                          zip area_name
    ## 5  90001 Florence/South Central (City of LA)          
    ## 6                   90002 Watts (City of LA)          
    ## 7           90003 South Central (City of LA)          
    ## 8            90004 Hancock Park (City of LA)          
    ## 9               90005 Koreatown (City of LA)          
    ## 10           90006 Pico Heights (City of LA)

    tail(zips_df)

    ##                                 zip area_name
    ## 486                  93560 Rosamond         X
    ## 487                  93563 Valyermo         X
    ## 488                 93584 Lancaster         X
    ## 489                 93586 Lancaster         X
    ## 490                  93590 Palmdale         X
    ## 491 93591 Palmdale/Lake Los Angeles         X

    ## How many rows have zip and area name merged?
    zips_df$zip <- as.character(zips_df$zip)
    zips_df$area_name <- as.character(zips_df$area_name)
    length(zips_df$zip[grep(" [aA-zZ]", zips_df$zip)])

    ## [1] 175

    ## For all those rows with zip and area name merged, split the text and replace
    zips_df[grep(" [aA-zZ]", zips_df$zip),] <- str_split_fixed(zips_df$zip[grep(" [aA-zZ]", zips_df$zip)], " ", n = 2)
    dim(zips_df)

    ## [1] 487   2

    ## Flag whether an area is in the "City of LA" (vs in-line text), to clean and lean things out
    zips_df$LA_city <- grepl("City of LA", zips_df$area_name, ignore.case = T)
    zips_df$area_name <- gsub("\\(City of LA\\)| \\(City of LA\\)", "", zips_df$area_name, ignore.case = T)

    ## Preview final data
    head(zips_df)

    ##      zip              area_name LA_city
    ## 5  90001 Florence/South Central    TRUE
    ## 6  90002                  Watts    TRUE
    ## 7  90003          South Central    TRUE
    ## 8  90004           Hancock Park    TRUE
    ## 9  90005              Koreatown    TRUE
    ## 10 90006           Pico Heights    TRUE

    tail(zips_df)

    ##       zip                 area_name LA_city
    ## 486 93560                  Rosamond   FALSE
    ## 487 93563                  Valyermo   FALSE
    ## 488 93584                 Lancaster   FALSE
    ## 489 93586                 Lancaster   FALSE
    ## 490 93590                  Palmdale   FALSE
    ## 491 93591 Palmdale/Lake Los Angeles   FALSE

#### Step 3. Geocode (lat, lon) L.A. County zips using Google’s Geocoding API

Unfortunately, this API is rate-limited, so we’ll factor that into our
design.

    if(!file.exists("dat/los-angeles-county_zip-codes_lat-lon.csv")){
      
        ## Grab my Google Geocode API key and register it with GGmaps, my easy interface to Google Maps family of APIs
        geo_key <- readRDS("geo_key.RDS")
        register_google(geo_key)
        
        ## Run zips through Geocode API call and keep lat/lon as new column values per zip
        zips_df[, c("lon", "lat")] <- geocode(zips_df$zip, output = "latlon")
        
        ## Save table for later use
        write.csv(zips_df, "dat/los-angeles_county_zip-codes_lat-lon.csv", row.names = F)
        
    } else {
      
        ## Let user know we've already got the data and therefore don't need to call the API
        print("Zip to Lat/Lon lookup table already exists. Skipping API call, then previewing data...")
      
        ## Get existing data, if it's there
        zips_df <- read.csv("dat/los-angeles-county_zip-codes_lat-lon.csv", stringsAsFactors = F)
      
    }

    ## [1] "Zip to Lat/Lon lookup table already exists. Skipping API call, then previewing data..."

    ## Preview the data
    head(zips_df)

    ##     zip              area_name LA_city       lon      lat
    ## 1 90001 Florence/South Central    TRUE -118.2468 33.96979
    ## 2 90002                  Watts    TRUE -118.2497 33.95111
    ## 3 90003          South Central    TRUE -118.2731 33.96580
    ## 4 90004           Hancock Park    TRUE -118.3082 34.07489
    ## 5 90005              Koreatown    TRUE -118.3097 34.05788
    ## 6 90006           Pico Heights    TRUE -118.2965 34.04708

    tail(zips_df)

    ##       zip                 area_name LA_city       lon       lat
    ## 482 93560                  Rosamond   FALSE -118.3228 34.884301
    ## 483 93563                  Valyermo   FALSE -117.7491 34.405338
    ## 484 93584                 Lancaster   FALSE -118.1400 34.700000
    ## 485 93586                 Lancaster   FALSE  110.3391  1.542587
    ## 486 93590                  Palmdale   FALSE -118.0600 34.500000
    ## 487 93591 Palmdale/Lake Los Angeles   FALSE -117.8194 34.592562

    ## Any NAs?
    print("Possible problem zips, for inspection:")

    ## [1] "Possible problem zips, for inspection:"

    zips_df[is.na(zips_df$lon),]

    ##      zip        area_name LA_city lon lat
    ## 79 90080 Airport Worldway    TRUE  NA  NA

#### Step 4. Send lat, lons to Native-land.ca API to get Native American lands and languages each area covers

For each lat, lon combo, we ping Native-land.ca’s API to get the Native
American lands that once

    ## Add empty territories and languages fields for now
    zips_df$native_territories <- NA
    zips_df$native_languages <- NA

    ## For each lat, lon combo, call the API
    if(!file.exists("dat/los-angeles-county_native-american-lands.csv")){
      
        for(i in 1:length(zips_df$zip)){
      
            if(!is.na(zips_df$lon[i])){
              
              
              api_resp <- GET("https://native-land.ca/api/index.php", 
                              query = list(maps = "languages,territories", position = paste0(zips_df$lat[i], ",", zips_df$lon[i])),
                              add_headers('User-Agent' = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36'))
              api_content <- fromJSON(rawToChar(api_resp$content))$properties
              zips_df$native_territories[i] <- paste0(api_content$Name[grep("territories", api_content$description)], collapse = ",")
              zips_df$native_languages[i] <- paste0(api_content$Name[grep("languages", api_content$description)], collapse = ",")
              
            } else {
              
              print(paste0("Not a valid set of coordinate values for area: ", zips_df$area_name[i], " (", zips_df$zip[i], ").", " Skipping."))
              
            }
        }
    } else {
        
        ## Let user know we've already got the data and therefore don't need to call the API
        print("Table already exists. Skipping API call, then previewing data...")
      
        ## Get existing data, if it's there
        zips_df <- read.csv("dat/los-angeles-county_native-american-lands.csv", stringsAsFactors = F)
      
    }

    ## [1] "Table already exists. Skipping API call, then previewing data..."

    ## Preview data
    head(zips_df)

    ##     zip              area_name LA_city       lon      lat  native_territories
    ## 1 90001 Florence/South Central    TRUE -118.2468 33.96979 Chumash,Tongva,Kizh
    ## 2 90002                  Watts    TRUE -118.2497 33.95111 Chumash,Tongva,Kizh
    ## 3 90003          South Central    TRUE -118.2731 33.96580 Chumash,Tongva,Kizh
    ## 4 90004           Hancock Park    TRUE -118.3082 34.07489 Chumash,Tongva,Kizh
    ## 5 90005              Koreatown    TRUE -118.3097 34.05788 Chumash,Tongva,Kizh
    ## 6 90006           Pico Heights    TRUE -118.2965 34.04708 Chumash,Tongva,Kizh
    ##   native_languages
    ## 1           Tongva
    ## 2           Tongva
    ## 3           Tongva
    ## 4           Tongva
    ## 5           Tongva
    ## 6           Tongva

#### Step 5. Deliver the Data Product (CSV format; can push to DB or API if/when required)

    ## Save to file
    write.csv(zips_df, "dat/los-angeles-county_native-american-lands.csv", row.names = F)

    ## List the unique tribes across LA county
    print("Unique Native American territories across LA County:")

    ## [1] "Unique Native American territories across LA County:"

    unique(na.omit(unlist(strsplit(zips_df$native_territories, ","))))

    ## [1] "Chumash"                           "Tongva"                           
    ## [3] "Kizh"                              "Micqanaqa’n"                      
    ## [5] "Fernandeño Tataviam"               "Acjachemen (Juaneño)"             
    ## [7] "Payómkawichum (Luiseño)"           "Yuhaviatam/Maarenga’yam (Serrano)"
    ## [9] "Kitanemuk"

[***Click to download lookup table of Los Angeles County Zip Codes to
Native American Territories and Languages
&gt;&gt;***](https://raw.githubusercontent.com/judecalvillo/native-land-attribution/master/dat/los-angeles-county_native-american-lands.csv)

------------------------------------------------------------------------

Thanks!  
-[Jude C.](https://www.linkedin.com/in/judecalvillo)

![](https://drive.google.com/uc?export=download&id=19O0LHWMrEzDmuVVwQFVBLQYGSf0S-siz)
