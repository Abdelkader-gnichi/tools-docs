Number of USP Agents         OVHCloud VPS (shared specs configartion) => 1 684,800 din HT/12 mois                         MongoDB Profile | ( Base Price x 3 )
<= 1,000                     VPS-6 / 24 vCores / 96 Go RAM / 400 Go SSD NVMe / 3 Gbit/s bande passante publique           M10      $0.24/hr ( $0.08/hr x 3 )
1,000 – 10,000               Same conficugartion                                                                          M20      $0.60/hr ( $0.20/hr x 3 ) 
10,000 – 50,000              Same conficugartion                                                                          M30      $1.62/hr ( $0.54/hr x 3 )
50,000 – 100,000             Same conficugartion                                                                          M40      $3.12/hr ( $1.04/hr x 3 )
100,000 – 300,000            Same conficugartion                                                                          M50      $6.00/hr ( $2.00/hr x 3 )
300,000 – 900,000            Same conficugartion                                                                          M60      $10.95/hr ( $3.95/hr x 3 )






datamodel: 50 kb | 0.5MB ( 50 kb * 10)
################################################## 
bootevents: 268B
DiagnosticsConfig: 347B
dignostics: 332B
executionlogs: 2KB
notifications: 581B
presets: 363B
provisions: 520B
statistics: 14KB
transformeddatas: 2KB
####################################################
Total in bytes:
268 + 347 + 332 + 2,048 + 581 + 363 + 520 + 14,336 + 2,048 = 20,843 bytes =  0.02 MB => 0.2 MB safe value multiplied by 10
#########################################################################################################
Total Device Storage Size at Onbording in the System:
0.2 MB + 0.5 MB = 0.7 MB
########################################################################### 
Total Device Storage Size after 7 days with all telemtry features are enabled:
0.2 MB * 288 * 7 days = 405 MB (0.4 GB) this value must be reset to 0.7 MB (initial device storage size during onboarding)  
0.02 MB * 288 * 7 days = 40.5 MB (0.04 GB) this value must be reset to 0.7 MB (initial device storage size during onboarding) <=> real value without 10 multiplication

reference value for 1000 connected devices:
0.4 GB * 1000 device = 400 GB per 1000 devices (all telemtry metrics are enabled without storage size reset) | 0.7 MB * 1000 devices = 700 MB = 0.7 GB per 1000 devices (very safe values * 10)

0.04 GB * 1000 device = 40 GB per 1000 devices  (all telemtry metrics are enabled without storage size reset) | 0.7 MB * 1000 devices = 700 MB = 0.7 GB per 1000 devices ( real values)

General Conclusion:
Use the 1000 devices reference value to make any device level scale calculation
