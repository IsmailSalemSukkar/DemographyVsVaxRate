library(tidyverse)
library(MuMIn)
library(DHARMa)
library(glmmTMB)
library(car)
library(emmeans)
library(viridis)
options(scipen = 9999)

CensusPercBlack <- 13.6
CensusPercHispanic <- 18.9
CensusPercAsian <- 6.1
CensusPercPacIsl <- 0.3
CensusPercNative <- 1.3
CensusPercWhite <- 59.3

#There is a common sentiment in the media that certain racial groups are more
#likely to reject the vaccine then others. Is this actually true?


setwd(dirname(rstudioapi::getActiveDocumentContext()$path))

#run the following ONCE 
vaxData <- read.csv("https://data.cdc.gov/api/views/km4m-vcsb/rows.csv?accessType=DOWNLOAD")

write.csv(vaxData, "vaxData.csv")
######Continue from here if not first time running script###
vaxData <- read.csv("vaxData.csv") 
vaxData<- vaxData[2:26]

vaxData <- vaxData[grep("Race_eth",vaxData$Demographic_category),]

vaxData <- vaxData %>% mutate(Demographic_category = str_remove_all(Demographic_category,
                                                         "Race_eth_"))


#Lets see for black americans if this holds any weight

t.test(vaxData[vaxData$Demographic_category=="NHBlack",]$Administered_Dose1_pct_known,
       mu=CensusPercBlack, alternative = "less")
#sig lower than Census Percentage

mean(vaxData[vaxData$Demographic_category=="NHBlack",]$Administered_Dose1_pct_known)/
  CensusPercBlack - 1

#~32% underperformance

#Hispanic

t.test(vaxData[vaxData$Demographic_category=="Hispanic",]$Administered_Dose1_pct_known,
       mu=CensusPercHispanic, alternative = "less")
#sig lower than Census Percentage


mean(vaxData[vaxData$Demographic_category=="Hispanic",]$Administered_Dose1_pct_known)/
  CensusPercHispanic - 1

#~2% underperformance

#Asian

t.test(vaxData[vaxData$Demographic_category=="NHAsian",]$Administered_Dose1_pct_known,
       mu=CensusPercAsian, alternative = "greater")
#sig greater than census percentage


mean(vaxData[vaxData$Demographic_category=="NHAsian",]$Administered_Dose1_pct_known)/
  CensusPercAsian - 1

#8% overperformance

#Asians appear to outperform their population percentage

#Pacific Islander

t.test(vaxData[vaxData$Demographic_category=="NHNHOPI",]$Administered_Dose1_pct_known,
       mu=CensusPercPacIsl, alternative = "less")
#sig less than census percentage

mean(vaxData[vaxData$Demographic_category=="NHNHOPI",]$Administered_Dose1_pct_known)/
  CensusPercPacIsl - 1

#2% underperformance


#Pacific Islander appear to be right on their census number

#Native

t.test(vaxData[vaxData$Demographic_category=="NHAIAN",]$Administered_Dose1_pct_known,
       mu=CensusPercNative, alternative = "less")
#sig less than census percentage

mean(vaxData[vaxData$Demographic_category=="NHAIAN",]$Administered_Dose1_pct_known)/
  CensusPercNative - 1

#22% underperfomance


#White

t.test(vaxData[vaxData$Demographic_category=="NHWhite",]$Administered_Dose1_pct_known,
       mu=CensusPercWhite, alternative = "less")
#sig less than census percentage



mean(vaxData[vaxData$Demographic_category=="NHWhite",]$Administered_Dose1_pct_known)/
  CensusPercWhite - 1

#2% underperfomance



#In a preliminary look, it does APPEAR that ethnicity does have an effect on 
#likelihood on vaccination 
#Lets find out for sure. 

vaxData$censusPerc <- 0

vaxData$censusPerc[vaxData$Demographic_category=="NHBlack"]<- 
  (vaxData$Administered_Dose1_pct_known[vaxData$Demographic_category=="NHBlack"]/
     CensusPercBlack) - 1

vaxData$censusPerc[vaxData$Demographic_category=="Hispanic"]<- 
  (vaxData$Administered_Dose1_pct_known[vaxData$Demographic_category=="Hispanic"]/
     CensusPercHispanic) - 1

vaxData$censusPerc[vaxData$Demographic_category=="NHAsian"]<- 
  (vaxData$Administered_Dose1_pct_known[vaxData$Demographic_category=="NHAsian"]/
     CensusPercAsian) - 1

vaxData$censusPerc[vaxData$Demographic_category=="NHNHOPI"]<- 
  (vaxData$Administered_Dose1_pct_known[vaxData$Demographic_category=="NHNHOPI"]/
     CensusPercPacIsl) - 1

vaxData$censusPerc[vaxData$Demographic_category=="NHAIAN"]<- 
  (vaxData$Administered_Dose1_pct_known[vaxData$Demographic_category=="NHAIAN"]/
     CensusPercNative) - 1

vaxData$censusPerc[vaxData$Demographic_category=="NHWhite"]<- 
  (vaxData$Administered_Dose1_pct_known[vaxData$Demographic_category=="NHWhite"]/
     CensusPercWhite) - 1

vaxDataAnalysis<- vaxData[!(vaxData$Demographic_category=="known") & 
                              !(vaxData$Demographic_category=="NHMult_Oth") &
                            !(vaxData$Demographic_category=="NHMultiracial") &
                            !(vaxData$Demographic_category=="unknown")&
                            !(vaxData$Demographic_category=="NHOther"),]


#summary(as.factor(vaxDataAnalysis$Demographic_category))

fitDemo <- glmmTMB(censusPerc~Demographic_category,data=vaxDataAnalysis)
plot(simulateResiduals(fitDemo))

Anova(fitDemo) #does appear that demographic is a sig factor

emmeans(fitDemo,~Demographic_category,type="response")

ggplot(data=vaxDataAnalysis,aes(x=Demographic_category,y=censusPerc))+
  geom_boxplot(aes(fill=Demographic_category)) + 
  scale_fill_viridis(discrete=T, option="turbo",begin = .25,end=.75) +
  xlab("Demographic Category") + 
  ylab("Vaccine % / Census % - 1") + 
  ggtitle("Vaccination Rates of Different Demographic Groups",
          "Black underperforms by ~32%, Native American underperforms by ~23%")

ggsave("demoVsVaxRate.png",width = 1920, height = 1080,units="px")

#We can see that black americans have the lowest perfomance when it comes 
#to getting vaccinated, with Native Americans coming in an a close second,
#while asian american have the strongest performance 

#So perhaps the annecdotal evidence the media presents does have some truth behind it
#More work needs to be done to protect and vaccinate these historically marginilized groups

#Flaws: ~33% of americans vaccinated we do not know their demographic group. For all we know,
#this may be the a decision black and native american groups decided to identify as. 





