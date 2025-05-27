 Travel_Tide: Traveler Segmentation Project

Overview
This project segments Travel_Tide’s user base into six distinct traveler personas by analyzing data from user, flight, hotel, and session records starting from January 2023. Our approach focuses on behavioral signals, supported by demographics when data is sparse.
Segmentation Approach

This analysis segments our user base into six distinct traveler personas by leveraging booking patterns, session activity, and demographic attributes drawn from user, flight, hotel, and session data since January 2023. Our primary classification was based on behavioral indicators such as trip count, session engagement, checked baggage, and hotel stays. Where behavioral data was insufficient, age was used as a fallback for segmentation.
                                               
Goal
To deliver a data-driven segmentation that allows the business to personalize experiences, increase engagement, and maximize customer lifetime value.  


| Tool                              | Purpose                                                    |
| --------------------------------- | ---------------------------------------------------------- |
| **SQL**                           | Data extraction, transformation, and metric calculation    |
| **Tableau**                       | Data visualization and persona dashboard creation          |


Methodology
 Data Sources

User Table: Age, gender,sign_up_date,has_children

Flight Table: base_fare_usd, seats, return_flight_booked, checked bags

Hotel Table: Nights, rooms, hotel_per_room_usd

Session Table: flight_booked, hotel_booked, page_clicks

 Segmentation Logic (SQL & Decison Tree Logic)

Primary classification:

Trip count

Checked baggage

Hotel nights

Session engagement

Fallback classification:

Age buckets used if behavioral data is insufficient



### 1.Cleaned_Sql_Query 2. Traveler Persona(User) Segmentation: Behavior-Driven Insights and a Fall_back 
Age-Based condition for Segmentation 3. Analysis of Customer Segments by Perks

```sql
1. WITH user_level AS (
   SELECT  s.user_id, COUNT(session_id)
	FROM sessions s
	GROUP BY user_id
	HAVING COUNT(session_id) > 7 ),
joined_tables AS (SELECT *,
  	CASE WHEN check_out_time::date - check_in_time::date < 1
  	THEN 1
  	ELSE check_out_time::date - check_in_time::date END as nights_new,
  	CASE WHEN rooms = 0 THEN 1 ELSE rooms END as new_rooms
	FROM sessions s
	LEFT JOIN users u
	ON s.user_id = u.user_id
	LEFT JOIN hotels h
	ON s.trip_id = h.trip_id
	LEFT JOIN flights f
	ON s.trip_id = f.trip_id
	JOIN user_level ul
	ON s.user_id = ul.user_id
  	   WHERE s.session_start >= '2023-01-04')
SELECT *
FROM joined_tables


2. SELECT
  u.user_id,
  u.gender,
  EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) AS age,
  u.sign_up_date,
  u.married,
  u.has_children,

  COUNT(DISTINCT s.session_id) AS total_sessions,
  ROUND(AVG(s.page_clicks), 2) AS avg_page_clicks_per_session,
  COUNT(DISTINCT f.trip_id) AS flight_count,
  COUNT(DISTINCT h.trip_id) AS hotel_count,
  COALESCE(SUM(h.nights), 0) AS total_nights,
  BOOL_OR(f.return_flight_booked) AS return_ticket,
  COALESCE(SUM(f.checked_bags), 0) AS total_checked_bags,
  COUNT(DISTINCT s.trip_id) AS num_trips,

  CASE
    WHEN u.has_children = TRUE
         AND COALESCE(SUM(f.checked_bags), 0) >= 4
         AND COALESCE(SUM(h.nights), 0) >= 7
         THEN 'Family Traveler'

    WHEN COUNT(DISTINCT f.trip_id) >= 5
         AND COALESCE(SUM(f.checked_bags), 0) >= 1
         AND COALESCE(SUM(h.nights), 0) > 5
         THEN 'Frequent Traveler'

    WHEN COUNT(DISTINCT f.trip_id) >= 3
         AND COALESCE(SUM(f.checked_bags), 0) = 0
         AND COALESCE(SUM(h.nights), 0) <= 3
         AND BOOL_OR(f.return_flight_booked) = TRUE
         AND AVG(h.nights) < 5
         THEN 'Business Traveler'

    WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 21 AND 30
         AND COUNT(DISTINCT f.trip_id) = 0
         AND COUNT(DISTINCT h.trip_id) = 0
         AND COUNT(DISTINCT s.session_id) >= 5
         THEN 'Dreamer'

    WHEN COUNT(DISTINCT s.session_id) < 8
         AND EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) < 21
         THEN 'Young Traveler'

    WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) >= 61
         AND BOOL_OR(f.return_flight_booked) = TRUE
         AND COALESCE(SUM(f.checked_bags), 0) >= 1
         THEN 'Senior Traveler'

    -- Fallback based on age
    WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) < 21 THEN 'Young Traveler'
    WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 21 AND 30 THEN 'Dreamer'
    WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 31 AND 40 THEN 'Business Traveler'
    WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 41 AND 50 THEN 'Frequent Traveler'
    WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 51 AND 60 THEN 'Family Traveler'
    WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) >= 61 THEN 'Senior Traveler'

  END AS persona_segment,

  CASE
    WHEN AVG(h.nights) >= 5 THEN 'Long Stay'
    ELSE 'Short Stay'
  END AS stay_type

FROM users u
LEFT JOIN sessions s ON u.user_id = s.user_id
LEFT JOIN flights f ON s.trip_id = f.trip_id
LEFT JOIN hotels h ON s.trip_id = h.trip_id

WHERE s.session_start >= DATE '2023-01-04'

GROUP BY
  u.user_id, u.gender, u.birthdate, u.sign_up_date, u.married, u.has_children

HAVING COUNT(DISTINCT s.session_id) > 7; 

3. WITH user_personas AS (
   SELECT
    u.user_id,
    u.gender,
    EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) AS age,
    u.sign_up_date,
    u.married,
    u.has_children,
    COUNT(DISTINCT s.session_id) AS total_sessions,
    ROUND(AVG(s.page_clicks), 2) AS avg_page_clicks_per_session,
    COUNT(DISTINCT f.trip_id) AS flight_count,
    COUNT(DISTINCT h.trip_id) AS hotel_count,
    COALESCE(SUM(h.nights), 0) AS total_nights,
    BOOL_OR(f.return_flight_booked) AS return_ticket,
    COALESCE(SUM(f.checked_bags), 0) AS total_checked_bags,
    COUNT(DISTINCT s.trip_id) AS num_trips,
    CASE
      WHEN u.has_children = TRUE
           AND COALESCE(SUM(f.checked_bags), 0) >= 4
           AND COALESCE(SUM(h.nights), 0) >= 7
           THEN 'Family Traveler'
      WHEN COUNT(DISTINCT f.trip_id) >= 5
           AND COALESCE(SUM(f.checked_bags), 0) >= 1
           AND COALESCE(SUM(h.nights), 0) > 5
           THEN 'Frequent Traveler'
      WHEN COUNT(DISTINCT f.trip_id) >= 3
           AND COALESCE(SUM(f.checked_bags), 0) = 0
           AND COALESCE(SUM(h.nights), 0) <= 3
           AND BOOL_OR(f.return_flight_booked) = TRUE
           AND AVG(h.nights) < 5
           THEN 'Business Traveler'
      WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 21 AND 30
           AND COUNT(DISTINCT f.trip_id) = 0
           AND COUNT(DISTINCT h.trip_id) = 0
           AND COUNT(DISTINCT s.session_id) >= 5
           THEN 'Dreamer'
      WHEN COUNT(DISTINCT s.session_id) < 8
           AND EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) < 21
           THEN 'Young Traveler'
      WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) >= 61
           AND BOOL_OR(f.return_flight_booked) = TRUE
           AND COALESCE(SUM(f.checked_bags), 0) >= 1
           THEN 'Senior Traveler'
      ELSE
        CASE
          WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 21 AND 30 then 'Young Traveler'
          WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 21 AND 30 THEN 'Dreamer'
          WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 31 AND 40 THEN 'Business Traveler'
          WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 41 AND 50 THEN 'Frequent Traveler'
          WHEN EXTRACT(YEAR FROM AGE(DATE '2023-07-28', u.birthdate)) BETWEEN 51 AND 60 THEN 'Family Traveler'
          ELSE 'Senior Traveler'
        END
    END AS persona_segment,
    CASE
      WHEN AVG(h.nights) >= 5 THEN 'Long Stay'
      ELSE 'Short Stay'
    END AS stay_type
  FROM users u
  LEFT JOIN sessions s ON u.user_id = s.user_id
  LEFT JOIN flights f ON s.trip_id = f.trip_id
  LEFT JOIN hotels h ON s.trip_id = h.trip_id
  WHERE s.session_start >= DATE '2023-01-04'
  GROUP BY u.user_id, u.gender, u.birthdate, u.sign_up_date, u.married, u.has_children
  HAVING COUNT(DISTINCT s.session_id) > 7
)

SELECT
  user_id,
  gender,
  age,
  sign_up_date,
  married,
  has_children,
  total_sessions,
  avg_page_clicks_per_session,
  flight_count,
  hotel_count,
  total_nights,
  return_ticket,
  total_checked_bags,
  num_trips,
  persona_segment,
  stay_type,
  CASE persona_segment
    WHEN 'Family Traveler' THEN 'Free hotel meal,free checked bags'
    WHEN 'Frequent Traveler' THEN 'Exclusive discounts'
    WHEN 'Business Traveler' THEN 'No Cancellation Fees, exclusive flight discounts'
    WHEN 'Dreamer' THEN 'Exclusive discounts on flights and hotels'
    WHEN 'Young Traveler' THEN 'Welcome bonus: 1 free night hotel with flight booking'
    WHEN 'Senior Traveler' THEN 'Priority boarding, free checked bag'
    ELSE 'Standard perks'
  END AS perks
FROM user_personas;



                                               
                                               Personas as follows

Frequent Travelers
Most engaged users, with the highest average trips and sessions.
High platform loyalty and booking frequency.
Recommended Perks: Exclusive discounts, loyalty rewards, and early access deals.

Family Travelers
Typically travel with children, book longer hotel stays, and check more bags.
Travel patterns show clear family-oriented needs.
Recommended Perks: Free hotel meals, kids-stay-free offers, and family room upgrades.  

Business Travelers 
Book short hotel stays with little to no checked luggage.  
Frequently opt for return flights, suggesting structured itineraries.
Recommended Perks: Free checked bags, flexible rescheduling, and priority  support. 


Dreamers 
High session counts but no bookings yet.  
Represent a large conversion opportunity.  
Recommended Perks: Exclusive first-time booking discounts, personalized travel inspiration, and targeted nudges.  

Young Travelers 
Under 21, low booking activity, early-stage users.  
Digital natives browsing frequently but booking less.  
Recommended Perks: 1-night free hotel stay with flight, referral rewards, and social media integrations.  

Senior Travelers 
Moderate platform usage, usually book return flights with checked baggage.  
Likely prefer convenience, comfort, and reliability.  
Recommended Perks: Free checked bags, priority boarding, and simplified booking flows.  


Methodology Note
Personas were primarily determined by behavioral metrics (e.g., number of trips, nights stayed, sessions, and checked bags). However, when these behaviors were inconclusive or minimal, users were segmented based on age brackets as a secondary rule to ensure comprehensive classification coverage.

Business Implications

-Tailoring perks and marketing strategies by persona will improve user satisfaction, drive bookings, and increase customer lifetime value.  

-Recognizing engagement patterns helps prioritize resource allocation, focusing on high-value segments such as Frequent and Family Travelers.  

-Converting Dreamers and engaging Young Travelers presents an opportunity to expand the customer base.  

-Senior Travelers benefit from comfort-oriented perks, enhancing their travel experience.  

