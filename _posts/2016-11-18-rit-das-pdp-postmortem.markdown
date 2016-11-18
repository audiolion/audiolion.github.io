---
layout: post
title:  "Website Downtime Postmortem"
date:   2016-11-18 15:05:00
author: Ryan Castner
categories: RIT DAS PDP
tags: das pdp rit department access services postmortem downtime website
cover:  "/assets/dev-log-background.jpg"
---

# Postmortem

Hello all! For those of you who do not know me my name is Ryan Castner and I am the web developer for the [Department of Access Services Website](https://daspdp.rit.edu).

On Thursday, November 17th, 2016 at approximately 12:05pm our website went down to fix an issue with deployments and lasted until Friday, November 18th, at 3:01pm. For the past two weeks we had not been able to push new changes to the website. Some of you are I am sure aware of the pains that caused, not being able to access your [dashboard](https://daspdp.rit.edu/accounts/dashboard) or having error pages on other parts of the website. ITS had an immediate opening to help us resolve the issue so we acted immediately.

That was a mistake.

We should have first communicated to everyone about the maintenance. The reason we did not communicate was because the disruption was expected to be a couple of minutes. A series of unfortunate events caused a downtime of over twenty four hours, however. Such a long disruption is inexcusable and we want to work hard to resolve the issues that caused this maintenance to snowball into a major outage. To do that we must first understand the precursors that led us to this event.

## Precursors

Two weeks ago we tried to deploy changes to the website in related to user feedback on the registration process. The deployment did not go as planned and my attempts to fix the deployment did not work. As a result we had to revert our site to the last known good state (which I had saved right before deploying). No data was lost, but this put our systems in an incongruent state with RIT's systems. As such we needed ITS intervation to resolve the issue.

The fix required two teams from ITS to coordinate with me as we resolved the error. After a conversation with one ITS team it was left to them to communicate the instructions to the other ITS team. Everything was verbal communication, nothing was written in the ticket except a later transcription of my words by the ITS team I spoke to.

## The Effects

When the greenlight was given to me signaling that the two teams had finished their work, all was good. However, I then noticed crucial files from our website were missing. Due to a miscommunication between myself and ITS, they had backed up the wrong files before taking down our site for the planned maintenance.

I immediately contacted them to inform them of the issue and a ticket was placed to get the correct backups. This event caused the large downtime. RIT's Systems team was unavailable to assist until Friday morning. When the files were restored on Friday I was able to stand the site back up and normal operations were able to continue. During the time I was standing the site back up, a few people may have noticed they could navigate to the site but they received an unauthorized error upon logging in. This was intentional, I wanted to block access to the site until I was sure everything was back up.

Due to us having to rely on backups, we had to revert to the nightly from 2016-11-16. Luckily no registration information should be lost because the registrations closed at 9pm that evening and the backup occurred after this. However, updated made on Thursday, November 17th, 2016 before noon will be lost. This likely only impacts managers who might have approved some registrations and will need to redo these actions. This is the most unfortunate consequence of the downtime. Data loss is something to not be taken lightly at all.

## Resolutions

The main problem arose from a failed deployment two weeks ago where we had to rollback the changes to the last known good state. We are know aware that if we proceed with such a rollback, due to the nature of our website, we will require ITS intervention to get back to a normal state.

The second problem is that we did not communicate to our Department about the potential downtime. In the future we intend to be much more transparent and informative about potential downtimes and maintenance.

The third problem is that we do not have very good error page messaging during site maintenance. In the future we are looking to create a better error page that will inform users that the website is undergoing maintenance currently and use a social media account to notify users about the current status and when the site will be back up.

The fourth problem is that there was poor communication between myself and ITS. In the future I want to be much more explicit and try to communicate better. I take responsibility for the incorrectly copied files. ITS simply followed my instructions, and they were not clear enough.

The final issue was data loss. There is a term called 'Disaster Recovery' in the IT world where a system is put in place should anything happen. We have some Disaster Recovery through ITS nightly backups, but we were also unable to access any of our data during this time. During the most recent PDP Committee meeting we discussed how we would go about implementing a Disaster Recovery system. We talked about having a separate website that provides read-only access to database information, a mirror of our website so if the website does go down, you will still be able to access information. That way if the site is down for an extended period of time, you can still access transcripts, schedules, certificates, event information, registration statuses, etc. while the site is being brought pack up. I will speak about this in future posts as we approach that time.

## Conclusion

The entire chain of events started two weeks ago when I tried to force a deployment to happen even after we hit roadblocks during the process. Instead of taking a step back, consulting with ITS and planning a new deployment, I forged ahead into the unknown. That unknown caused an emergency rollback as the changes did not apply correctly. The rollback caused our system to go out of sync with RIT's systems, and the maintenance and failure on my part to communicate led to the downtime.

We learn every day, as I am sure you all know, computer programming is a life-long learning skill as is American Sign Language. I have learned a great deal through this process and have many resolutions to now work on to ensure something like this does not happen again, or if it does, we are as prepared as possible for that situation.

Thank you all for your patience and understanding during these past couple days. Now it is time to look to the horizon.

Ryan Castner
