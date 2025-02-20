+++
author = "mike"
title = "Apolis"
date = "2025-02-19"
description = "Apolis launch blog post"
tags = [ "react", "typescript", "tanstackquery", ]
categories = [ "frontend", ]
series = ["App Launches"]
aliases = [""]
+++

This past month I've been very busy. I dove head first into an open source, greenfield project that sounded like it would make an immediate improvement in peoples' work helping a vulnerable part of our community.

The [Harry Tompson Center](https://www.harrytompsoncenter.org/) in New Orleans provides crucial day services for unhoused folks. Every year they facilitate 35,000 visits for over 4,000 unique individuals. With those numbers, their clipboard management process couldn't keep up.

Apolis is the solution we made for the HTC.

Our Project Manager Blake Bertuccelli-Booth linked up with the HTC with impeccable timing: the Super Bowl was ramping up here in New Orleans and Gov. Jeff Landry just made a hall-of-fame myopic decision to waste millions of dollars on temporarily relocating unhoused folks away from the eyes of the wealthy football fans. In the same breath he complained about ineffective methods New Orleans has used in the past for reducing the unhoused population. This isn't an original idea, I heard the same thing happened in Las Vegas for a Super Bowl.

Anyways, some genuinely materially beneficial work could be done with an organization like the HTC who has boots on the ground in helping the unhoused.

Some specific pain points we aimed to address:
- knowing who is waiting for a service and already received a service
- accurately tracking unique guests according to name and birthday across many visits
- communicating to guests upon sign-in that there is some sort of notification for them

Additionally, from the data collected over time, we will be able to see which services are most correlated with getting an unhoused person housed and stable.

## Planning

Can this all just be done in a spreadsheet? Are we over-engineering if we make a custom app? A spreadsheet could be a viable route if there is continuity in staff and volunteers at the center who are also versed in Excel. There is no such guarantee of who would be creating and analyzing the data long term, so we decided a web app would be the best way to go. Accessing the app via web browser made the app cross platform, volunteers can use any device to access it, and we could tailor it to make their workflows as easy as possible without being constrained by a proprietary software like Excel or Airtable.

As the initial team started to take shape, we landed on a Node.js tech stack with React on the frontend and a serverless (AWS Amplify) API interfacing with a Postgres database.

## Development

Being more familiar with React than backend serverless architecture at this point in my career, I filled a frontend dev role on this project. Ian Painter jumped in as well and put in a lot of incredible work.

After designing the wireframes for each view and component, we divvied up views to code, tracking our progress in Github issues.

One of my views is a beast: the service details page contains the guests queued for a service, guests who have completed the service, and if the service resource is limited at any given time, like showers, there is a slotting system where a user can manage the resources as the guests use them.

![](/img/service-details.png)

As services are provided daily, the users (staff/volunteers) at the HTC will be using this view hundreds of times a day. We don't want to make a user click any more than necessary as they juggle their fast paced environment. Only the leanest possible options will be available to the user.

## Biggest Challenges

### Async State Management
The service details page is heavily interactive. Users will be constantly updating the statuses of each guest using a service. Two example workflows:

- Non-resource-limited service like free shelter vouchers:
	- a guest requests a shelter voucher upon sign in, they are immediately queued to receive the service
	- the volunteer coordinating the vouchers will call their name when they are at the top of the queue, give them a voucher, then mark the guest as completed
- Resource-limited service like showers:
	- a guest requests a shower upon sign in, they are immediately queued to use a shower
	- the volunteer coordinating the service will call their name and assign them a shower
	- the volunteer will monitor the time they spend in the shower, referencing a start time that marks when they were assigned the shower
	- when the allotted time is up, the volunteer marks the guest as completed and the shower is freed up for another guest

These are two ideal workflows, but in a very active environment there will likely be missteps and changing circumstances, so at any given time the user needs to be able to update the guest to any status regardless of where in the ideal workflow they are.

Technically, since all the data is coming from the API, multiple requests need to be sent and many different scenarios need to be handled depending on what status the guest needs to be changed to and from. I could see the API calls and helper functions piling up in my components, it'd be awesome to avoid all that manual refetching as guest statuses change.

Ian implemented TanStack Router for our SPA and it was serving us very well, probably the best decision we made on this app, thanks Ian! TanStack also has Query (legacy name: React Query), and when I read about what it does, it sounded like the exact solution I needed for all this async state management and API querying.

#### How TanStack Query works and how I used it to my advantage

TanStack Query provides a QueryClient that can be accessed from anywhere in the app, similar to React's Context. Wrapping the app close to the root will be the QueryProvider which takes in the QueryClient as a prop.
```jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient();

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      ...
    </QueryClientProvider>
  );
}
```
I needed to leverage the QueryClient for services, so in the Services view component, I defined the queries that I would need to fetch the guest data organized by status. I have a `guestsSlotted`, `guestsQueued`, and `guestsCompleted` query to fetch the guests with those respective statuses. On page load, these three queries run in parallel and I get the guests I need for each table and card element.
```js
const { data: guestsSlotted, isPending: isSlotsPending } = useQuery({
  queryFn: () => fetchServiceGuestsSlotted(service.service_id),
  queryKey: ["guestsSlotted"],
  enabled: !!service.quota,
});

const { data: guestsQueued, isPending: isQueuedPending } = useQuery({
  queryFn: () => fetchServiceGuestsQueued(service.service_id),
  queryKey: ["guestsQueued"],
});

const { data: guestsCompleted, isPending: isCompletedPending } = useQuery({
  queryFn: () => fetchServiceGuestsCompleted(service.service_id),
  queryKey: ["guestsCompleted"],
});
```
Note the `queryKey` on each query, this is the tag that is registered with the QueryClient for this specific query, I can later use them to refire specific queries when needed.

So far, nothing special. I did not need TanStack Query for fetching the data on page load. I could easily replace these queries with the functions I passed in as the `queryFn`. But I need to update the guest data often and I absolutely need to avoid page refreshes.

The next crucial capability of TanStack Query is the mutation query. Where the `useQuery` hook is best used for basic fetching/GET requests, `useMutation` facilitates updating and refetching so that the manual refetching I was dreading is avoided.

```js
const { mutateAsync: moveToCompletedMutation } = useMutation({
  mutationFn: (guest: GuestResponse): Promise<number> =>
    updateGuestServiceStatus("Completed", guest, null),
  onSuccess: () => {
    queryClient.invalidateQueries();
  },
});
```

I can use `useMutation` anywhere in my component tree and reference specific queries I may want to invalidate, or I can invalidate them all as I have done above. When the mutation function is successful, I invalidate the three original queries which triggers them to fire again and now I have the most up-to-date information for all the guests in every status without needing to load the page again and without having to use extraneous state variables or prop drilling.

### Constraints on the available slot options for a queued guest

On a resource limited service, there cannot be two guests assigned to the same slot at the same time. To avoid user error, I needed to constrain the available slot options in the select dropdown. This meant that I needed to deduce the options from the currently occupied slots AND the current user intentions.

![](/img/assign-button.png)

I could use the information I had from the `guestsSlotted` query and that would get me half way there. Out of 10 slots, if there were guests occupying slots 1, 2, 3, the user would only be able to choose slots 4-10. But currently that means the user would have those options for every guest and could choose the same slot for two or more guests.

At this point, when a user clicks assign, the guest status is changed from queued to slotted and moves to the chosen slot, an API query is made. A user chooses a slot number in a drop down for at least one guest before the Assign button is active.

I originally had an `availableSlots` query with my `guest<status>` queries, which worked for the page load but in order for that to be responsive like my guest queries, I would have to send a request or mutation every time the user chose a slot. That was obviously inefficient.

However, here was the perfect case to use a state variable to track what slots were available. I could initialize it with an API call but then only update this client-side state variable as the user selects which slots they want to assign to users. Only after all their choices are made, with the state variable tracking them all, will they then click the assign button and all the guests will be mapped to their respective slot assignments.

To get over the last little bug while implementing it, I started an impromptu mob debugging session with the job hunters group at Operation Spark. It was a fun experience: with some help along with explaining aloud what my problem was, I was able to pinpoint why the user's slot choice wasn't resetting if they clicked an already chosen dropdown. Shout out to Chrome Devtools as well.

### Assigning multiple guests at a time

My fellow frontend dev Ian concluded that the biggest lesson he learned from this project is that when you're running into state management and reactivity issues in React, it's probably best to factor out part of the problem component into another component. I concur. Employing this strategy helped me out more than once on this project as well.

Originally, instead of the one assign button on top of the "Queued" table, I had an assign button next to every slot select dropdown. Before handing the app off to the HTC, almost immediately we found that we couldn't assign multiple people at once, the state would get lost and not update as intended.

I was keeping track of the slot selection state in the QueuedTable component, but since every row in the table had a dropdown and assign button, I couldn't track the state of each dropdown efficiently. I could pass an update function down as a prop but then I'd also have to change the state variable from a single value to an array of values. That's maybe ok, but then the state would change or reset every time I clicked any of the assign buttons.

Blake suggested having only one Assign button that would work for them all. When he said that, it felt so obvious. Reducing the many Assign buttons to one was the first step to solving this issue and the UI/UX would be much less noisy.

I tried solving the state management problem without factoring out the table rows into their own components, but it did not get me any further than I was. Factoring out the components made tracking the state of each row much easier. I'm recognizing that state with the most specific scope as possible will reduce the number of re-renders, leading to a much more optimized application overall.

So with a combination of reducing the Assign buttons to one single button, changing the state variable to an array of values that could be updated from each row component, and using a helper function to map all the state values to the respective slots, I was able to provide the user the ability to assign multiple people at once.

## Launch and Maintenance

We launched Apolis last week (Feb 11, 2025) and they have already made us aware of bugs and desired enhancements.

This is the first app I've been a part of launching directly to users who are immediately going to use it. I'm finding the feedback super helpful. As the developer, I'm not going to be using the app like an actual user with the actual use case but I want to make sure the app is optimal for that use case. I'm honestly just pumped I can receive the feedback and readily know how to implement the changes (or find out how). I'm very proud of my progress in the last two years of self-educating, bootcamp grinding, and now open source contributing.

I plan on being part of the maintenance of Apolis. There may be other use cases yet with other charity and solidarity organizations which I'm excited about. Please reach out if you want to contribute, or directly check the repo out [here](https://github.com/1111philo/apolis-app).

If you've made it this far, thanks for reading! Hire me? I'm looking for my breakout role in software development!