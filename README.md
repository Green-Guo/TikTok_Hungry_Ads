# WFM Smart Routing POC Draft 1 ReadMe 

### *Current as of 10/13/2023*

## Content: 
- Instructions
- Introduction 
- Part1: Scoring Models 
- Part 2: Matching Algorithm 
- Part 3: Result Analysis 
- Part 4: Future Work 
- Part 5: References

## Instructions:

In this working directory we have three notebooks ads.ipynb, moderator_score.ipynb, and WFM Smart Routing POC.ipynb

The main modeling scripts are in the self-contained WFM Smart Routing POC.ipynb consisting of two models for matching ads to their reviewers, as well as their respective results analysis of key metrics, such as the score difference (between ad and moderator), market similarity score, and the task handling time etc.

ads.ipynb and moderstor_score.ipynb: Data preprocessing and calculation of ads and moderators could be found in these 2 notebooks. The detailed weightage assignment and justification can also be found here. 
They process sample_data.xlsx (a data download for POC study consisting of 40,680 ads and 1,415 moderators' performance random sampled at Sep 9 from August'23 historical data) and output 6 csv files, which are ads and moderators split into good, medium, and bad categories (justification follows). 


## Introduction:

In this README file, we will provide an overview of our project, which focuses on optimizing the moderation process for social media advertisements. It consists of three key components: scoring models, namely a ads content scoring model and a moderator scoring model, a matching algorithm which assigns the highest-scoring ads to top-performing moderators; and plans for Further Work to continually improve and refine the moderation process.  

Using the scoring model, we were able to split the ads and moderators into 3 categories, with high, medium, and low scores.

We then explored two algorithms to have them assigned to the moderators of the same category, as shown in Figure 1.
<img width="593" alt="Screenshot 2023-09-09 at 11 18 43 PM" src="https://github.com/liuchennn1414/TikTok_Hackathon/assets/77218431/ce0698eb-f76f-4c18-83a4-deb6d98dfdf8">
Figure 1. Overview of algorithm 1

Another assignment algorithm is defined as Figure 2.

<img width="400" alt="Screenshot 2023-09-09 at 11 18 43 PM" src="https://github.com/Green-Guo/git_home/blob/master/screenshot-20231016-184241.png">
Figure 2. Overview of algorithm 2


Part 1: Scoring Models 

The first part focuses on the Scoring Models for ads and moderators. The score of an advertisement reflects its priority to be reviewed, while the score of a moderator reflects his performance.

A holistic round of data exploration & engineering has been conducted to fill up/remove missing values, visualize distribution, normalize highly skewed data (e.g. handling time from moderator) and standardize the scale of attributes to prevent latent weight. Correlation analysis has also been conducted to study potential correlation among attributes. 

### 1.1 Ads Content Scoring Model
How did we determine the “priority” of an advertisement? We designed 4 metrics stemming from the information given in our test/sample dataset, which assess advertisements in 4 distinct aspects: risk, profitability, urgency, and complexity. 

a. Risk
This component evaluates the potential risk associated with the content. Risk takes into account of punish_num and the number of days after the latest punished date. The more punishments the ad supplier has and the more recent the latest punishment is, the higher the risk of this content. 

b. Profitability
Profitability assessment helps us determine the financial viability and potential returns associated with the content. It ties with the avg ads revenue and ads revenue fields of the dataset. 

c. Urgency
Urgency scoring assesses how time-sensitive the content is. It helps in prioritizing and addressing content that requires immediate attention. Urgency is calculated using the start time, the earlier the ads need to be rolled out, the less time left for moderators to complete their review, and thus would be more urgent.

d. Complexity
The complexity component evaluates how intricate or involved the content is. It helps in understanding the level of effort required to deal with the content effectively. Complexity is given by the baseline_st field of the dataset.

The final scoring model for advertisement is as such: 
Y = 0.35*risk + 0.35*profitability+ 0.15*urgency + 0.15*complexity)

These 4 metrics are then assigned different weights in a linear model to co-determine the priority of the ad to be reviewed by the top-performing moderator. The higher the risk, higher the profitability, higher the urgency and higher the complexity, the better the moderator to be assigned.


### 1.2 Moderator Scoring Model
The Moderator Scoring Model is designed to evaluate the performance of moderators using a linear model based on 2 components: Accuracy and Efficiency. A justified weight is given to each component based on the business context. 

a. Accuracy
Accuracy scoring assesses how well moderators are in correctly identifying and categorizing content. It depends on the original Accuracy column given in the dataset.

b. Efficiency
Efficiency scoring evaluates how efficiently moderators handle the content. It combined the effect of productivity, utilisation and handling time using a linear model with justified weights. 

The final scoring model for moderator is as such: 
Y = 0.5Accuracy + (0.33Productivity + 0.3Utilisation - 0.33HandlingTime)

Please refer to the notebook for the details on how we generate the score and how we design the weights.


## Part 2: Matching Algorithm

### 2.1 An Objective Function Based Approach

In order to ensure that ads are efficiently matched to a moderator with a similar score, we split the ads and moderators into 3 categories, namely the high scoring category, average scoring category and low scoring category. The assignment of ads in these batches are done in a parallel manner. This ensures that high scoring ads are not handled by low scoring moderators and vice versa.

Within each category, ads are sorted according to the score, so that high priority ads are moderated first. They are then assigned to moderators in batches. A one-to-one assignment is done in each batch, and we aim to minimize the overall cost of assignment in each batch. The overall cost is defined by the loss function, described in details below. 

The loss function comprises of 3 components: 
The absolute difference between the scores given a pair of ad and moderator.
This measures the fitness between the ad and moderator in terms of scores
The maximum similarity between the ad’s country and moderator’s scope of countries:
In order to assign ads to the moderator with similar cultural and language context, we prompted ChatGPT to generate the similarity between each pair of distinct countries based on their official language used as well as the similarity of their cultural background. The similarity score varies from 0 to 1, and the closer to 1, the more suitable the moderator to review this ad.
As a moderator can review ads from more than 1 country, we only consider the country in a moderator’s scope that is the most similar to the country the ad came from.
This ensures that moderators handle ads that he/she is more familiar with
The length of task at hand of a moderator after he/she is assigned an ad.
By minimizing the time taken for a moderator to finish reviewing all ads assigned after each batch, we shorten the time taken to finish reviewing all the ads.

The 3 losses are normalized in each batch and assigned an individual weight, with the weight of the second cost being negative and the rest being positive. These values then sum up to be the overall loss of assignment in each batch.

The min-cost-assignment problem for each batch is handled with the Hungarian algorithm, also known as the Kuhn-Munkres algorithm, which is an optimization algorithm used for solving the assignment problem.

### 2.2 A Business Logic Centered Approach

We also considered another methodology as the business XFN preferred and more tied to their operational experience and SOP, be it a top task assigning to best reviewer both descending until (task) exhaustion procedure. Since we now have the ads and moderator pool both scored and classified, we will rank them from top to bottom and assign the highest priority task to highest scoring reviewer.

But instead of a simple task A to review 1, we must not give any task to reviewers who are stuck on their previously assigned task. Thus we must introduce a clock simulation. Since most tasks are finished in minutes (baseline st≤3), we set the tentative assignment interval to be 1 minute.

At the zero-th minute we map highest priority tasks as many as the number of moderators being tested in this modeling because at this initialization stage all moderators are considered free (or just started the day);

Then we advance the 'clock' by 1 interval to check which moderators will have theoretically finished using the standardized task type time; and we only give the already finished thus currently available reviewers new tasks (making sure we give the highest scoring in the new pool - sampling without replacement);

We keep advancing the imaginary clock until all tasks have their corresponding reviewer. As far as we know, this is a methodology most in line with business understanding and operationalization.


## Part 3: Model Result 
We will evaluate our matching model’s result via 3 perspectives: Score Difference, Market Similarity and Task Handling Time. Here we present the result for Good (highest priority ads among the 3 categories) segment as we most care about whether they're more optimally triaged.

Based on preliminary readout, we believe assignment mechanism/model 2 to be superior considering its outperformance vs. model 1.

<img width="597" alt="Screenshot 2023-09-09 at 11 19 33 PM" src="https://github.com/Green-Guo/TikTok_Hungry_Ads/blob/main/mode_result.png">

* a word of caution: Model 1's task processing time has bloated vis-a-vis random distribution, and we believe root cause is likely that high-score tasks are generally more complex, thus requiring a longer duration to moderate effectively. Despite this, we remain convinced that assigning complex tasks to high-score moderators is essential. We prioritize this approach because the risks associated with allowing potentially harmful content to pass or wrongly rejecting high-value content are considerably greater than the inconvenience of slightly longer wait times for advertisers. We have undertaken efforts to optimize both facets to the best of our ability.

** another word of caution: the processing time here is unreasonable but the only one we can use - baseline standard task as time measurement for hypothetic task review time under different task-mod matching schemes. It is literally impossible to estimate how much time a new moderator will take when looking at a historical task, should it be distributely differently than the fait accompli. We cannot use the historical real HT either. We only know by re-assigning tasks to a higher tier reviewer the turn around time should be lower - but cannot guess by how much. This is an active area of research and we are brainstorming with folks in the project team including PM and RD. After brainstorming with Yiran, we think it's possible to report how many tasks a model bumped up to better tiered moderators as it will theoretically result in better results so this will be added soon.


### 3.1 Score Difference
<img width="597" alt="Screenshot 2023-09-09 at 11 19 33 PM" src="https://github.com/liuchennn1414/TikTok_Hackathon/assets/77218431/b93aee92-fa2e-42d9-b227-c4c1f0c0b270">

Based on the optimized distribution of score difference between moderator and ads, the distribution has shifted left as compared with the random mean & distribution. It is a good indicator that our matching model reduces the score difference between moderator and ads, and good content is moderated by good moderators to make sure that those risky and highly profitable materials are handled appropriately. As this model 1 used min(score diff) as one of its objective, thus it performed better than model 2 in this area.

### 3.2 Market Similarity 
<img width="596" alt="Screenshot 2023-09-09 at 11 19 49 PM" src="https://github.com/liuchennn1414/TikTok_Hackathon/assets/77218431/c74e4678-e7fd-4a64-bd69-997513b0340c">


The improved market similarity score has been shifted to the right, and the median of average score has increased from 0.7 to 1.1 (a ~60 percent increase). As a result of our model, moderators are able to focus on moderating content which they are familiar with (from a linguistic and cultural perspective), which would improve their efficiency and accuracy, which has resulted in the optimization of both revenue and review time. Model 2 didn't treat this area in modeling.

### 3.3 Task Handling Time 
<img width="601" alt="Screenshot 2023-09-09 at 11 19 56 PM" src="https://github.com/liuchennn1414/TikTok_Hackathon/assets/77218431/55c5d60a-c7ca-477b-9621-0223fa9f4366">

As mentioned above, we noticed a pattern with higher handling time wrt. model 1, and we found out that this is contributed by the good moderator (i.e. the high score category).

The root cause is likely that high-score tasks are generally more complex, thus requiring a longer duration to moderate effectively. Despite this, we remain convinced that assigning complex tasks to high-score moderators is essential. We prioritize this approach because the risks associated with allowing potentially harmful content to pass or wrongly rejecting high-value content are considerably greater than the inconvenience of slightly longer wait times for advertisers. We have undertaken efforts to optimize both facets to the best of our ability. Model 2 however is reliable in reducing processing time however we mentioned this measurement is iterating.

## Part 4: Future Work

<img width="624" alt="Screenshot 2023-09-09 at 11 20 03 PM" src="https://github.com/liuchennn1414/TikTok_Hackathon/assets/77218431/4cc333a3-fab4-4316-9d98-c05a28cfb712">

1. We got word of a "model risk score" generated by machine moderation model for the ads involved in moderation tasks. Yiran is checking with Yiyu Li (who is the PM for smart assign/routing etc. under Tianyi) to get this score to be an input for this model. By understanding the content risk level of the advertisement, we're confident that our advertisement scoring model will yield more accurate results, in addition to simply relying on their advertiser's punishment record which could mask a lot of useful signal.

2. There are serious trouble when using baseline standard task as time measurement for hypothetic task review time under different task-mod matching schemes. It is literally impossible to estimate how much time the new moderator will take when looking at a historical task, should it be distributely differently than the fait accompli. We cannot use the historical real HT either. We only know by re-assigning tasks to a higher tier reviewer the turn around time should be lower - but cannot guess by how much. This is an active area of research and we are brainstorming with folks in the project team including PM and RD.

3. Last but not least, we need to reconcile the iteration epoch with smart routing rerank granularity (e.g. 1 minute), as it will impact the final design of matching algorithm. Suppose every minute smart routing re-ranks all the tasks in queue, then it will place a heavy burden on server capacity; however if we wait 10 minutes, there can be a large swath of idle time for fast reviewers lowering utilization. When put to RD they retorted with a nonsensical response "At present, task sorting and allocation should be the responsibility of the scheduling platform. Only after the auditor manually clicks on the 'assign me a task' on review platform will task allocation be triggered. Task sorting will give the task a 'task expected output time' when the task is reviewed, as the priority of task sorting." We are following up to see the inner workings of routing platform, as well as studying efficiency impacts under varying refresh interval schemes. After all, if we re-rank queueing tasks after a stochastic interval, ie. once any one finishes previous task and clicks for a new ad, the model will not be viable and will need to essential re-rank along with the pace of individual reviewer(s) beating the purpose of smart ranking which was supposed to globally see which tasks fit which reviewers.

4. unrelated to modeling itself, but we kicked off experiment design including sample sizing, site selection and planning with ops to make the experiment cycle planned for before Thanksgiving, so that there can be results when returning from recess.

## Part 5: Resources
1. Stochastic dynamic matching: A mixed graph-theory and linear-algebra approach: https://arxiv.org/pdf/2112.14457.pdf
2. A framework for dynamic matching in weighted graphs: https://zlangley.com/a-framework-for-dynamic-matching-in-weighted-graphs.pdf





