```
library(plyr)

toys <-c('Psyduck','Pikachu','Pikachu Car')
totaltoys <- length(toys)
prob = NULL

#use this for equal probablity for each toy 
for (w in 1:length(toys))  {
  prob[w] <- 1/totaltoys
}



done=0
open<-NULL
box<-0

#calculates from j number of samples
for (j in 1:500){
#buys 100 meals at most to obtain all the toys, but will break after all toys obtained.
# may need greater value i for more toys
  for ( i in 1:100) {
    box[i]<-sample(toys, 1, replace = TRUE, prob)
    coun<-count(box)
    if (nrow(coun)==totaltoys)
      break
    
  }
  openbox<-sum(coun$freq)
  open<-rbind(open,openbox)
  box<-0
}  
meanopen<-mean(open)
meanopen
sdopen<-var(open)^.5
sdopen

```
