### Introduction

Book my show is a ticket booking application which helps users in booking single/multiple seats for a movie/show/event etc. in a particular geographical location. It is possible that multiple users try to book the same show and sometime same seat at the same time (concurrently). So we need to design a system that can handle such concurrent requests, so that no two person has confirmation of same seat booking.

### Ways to handling concurrent booking requests

When handling such concurrent requests, there are multiple approaches to solve the concurrency issue and provide a great user experience. We would be seeing some of the mostly used approaches not only in book my show but other ticket booking applications too.

#### DB Locking

Once the user selects the available seat, application locks the seat temporarily for the next 10 minutes. The application interact with the DB and block the seat for the user. The ticket will be booked temporarily for the current user and for all the other users the seat will be unavailable for the next 10 minutes. If the user fails to book the ticket within that timeframe the seat will be released for the other users. This should be done on a first come first serve basis.

![Seat_Booking.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649595959237/bSxe3ZX9B.png?auto=compress,format&format=webp)

To maintain/lock a row in DB, we need to add a timeslot(starttime and endtime) columns and isReserved flag(successful booking can be denoted by **true**) in our table.

**Algorithm**

*   User U1 initiates the booking, we set the start time to _T1_ and end time to _T3_(T1+10)
*   User U2 trying to access the same temporarily booked seat between _T1_ and _T3_ will not be able to book a seat.
*   If booking by U1 is successful between _T1_ and _T3_, we will toggle booked flag to **true** to ensure final booking confirmation.

![BMS_DB.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649611868223/_gjuOyjVk.png?auto=compress,format&format=webp)

#### Transaction Isolation Level

We can use transactions in SQL database to avoid any clashes. In SQL servers, we can utilize [Transactional Isolation levels](https://radhikanarang.hashnode.dev/transaction-isolation-levels-in-dbms) to lock the rows before we update them. We can use _Serializable_ which is the highest isolation level and guarantees safety from:

*   Dirty reads
*   Non-repeatable reads
*   Phantom reads

With above database transaction, we can safely assume that the reservation will be marked successfully and no two customers(users) will be able to reserve the same seat.

**Algorithm**

*   We set Transaction level of DB to TRANSACTION_SERIALIZABLE
*   If some user requests for booking seats, Transactional Isolation level locks the rows before we update them.
*   Now, if another user has requested a booking of 2 seats _S1_, _S2_, we check if these seats are locked
*   We check, if the count of unlocked/available seats among the requested seats is equal to the number of seat booking requested by user (here 2).
*   If the above condition is **true**, then only we allow booking (whole transaction) to proceed.
*   If the above condition is **false**, then the booking (whole transaction) is cancelled.

    public boolean makeBooking(Booking booking){
        List<Seat> seats = booking.getSeats();
        Integer seatIds[] = new Integer[seats.size()];
        int index = 0;
        for(Seat seat:seats){
            seatIds[index++] = seat.getSeatId();
        }

        Connection dbConnection = null;
        try{
            dbConnection = getDBConnection();
            dbConnection.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);

            statement st = dbConnection.createStatement();
            String selectSQL = "SELECT * FROM Seat where showId = ? && seatId in (?) && isReserved=0";
            PreparedStatement preparedStatement = dbConnection.prepareStatement(selectSQL);
            preparedStatement.setInt(1,booking.getShow().getShowId());
            Array array = dbConnection.createArrayOf("INTEGER",seatIds);
            preparedStatement.setArray(2,array);
            // all the seats being checkout out by other users will be locked and so will not be part of resultSet
            ResultSet rs = preparedStatement.executeQuery();

            if(rs.next()){
                rs.last(); //checking the last row
                int rowCount = rs.getRow(); // count of rows which are not locked

                if(rowCount==seats.size()){ // if no seats are locked
                    //update Seat Table
                   // update Booking Table
                  dbConnection.commit();
                  return true;
                 }
            }
        }catch(SQLException e){
            dbConnection.rollback();
            System.out.println(e.getMessages());
        }
        return false;
    }

#### Synchronisation with distributed caching

In the above code, if we notice the actual booking logic, is when the system gets the booking requests and tries to update the DB with the booking details. So, we can consider this code block to be the _critical section_ of our ticket booking function.

Now, if we can restrict only one process(thread) to access the critical section, we would be able to handle concurrent(parallel) requests successfully. To do the same, we can use a synchronised block that locks the critical section allowing only one thread to execute the code inside this section.

    public boolean makeBooking(Booking booking){
        List<Seat> seats = booking.getSeats();
        Integer seatIds[] = new Integer[seats.size()];
        int index = 0;
        for(Seat seat:seats){
            seatIds[index++] = seat.getSeatId();
        }

        Connection dbConnection = null;
        try{
            dbConnection = getDBConnection();
            dbConnection.setTransactionIsolation();

            statement st = dbConnection.createStatement();
            String selectSQL = "SELECT * FROM Seat where showId = ? && seatId in (?) && isReserved=0";
            PreparedStatement preparedStatement = dbConnection.prepareStatement(selectSQL);
            preparedStatement.setInt(1,booking.getShow().getShowId());
            Array array = dbConnection.createArrayOf("INTEGER",seatIds);
            preparedStatement.setArray(2,array);
            // all the seats being checkout out by other users will be locked and so will not be part of resultSet
            ResultSet rs = preparedStatement.executeQuery();

            synchronized(this){ // if no seats are locked
                    //update Seat Table
                   // update Booking Table
                  dbConnection.commit();
                  return true;
             }catch(SQLException e){
                dbConnection.rollback();
                System.out.println(e.getMessages());
             }
             return false;
    }

The above synchronisation, works fine when we have only one instance(one JVM), but this would not work in case we have multiple instances(multiple JVM's) because a synchronisation block wouldn't provide access control to a shared resource within one JVM, but not across JVM's.

To tackle the same in distributed systems, we need to use a distributed cache like Redis or HazelCast that can read and update the value in one atomic operation. In hazelcast we can do the same using EntryProcessor(similar to what we did in [EntryProcessor-Rate Limiting Algorithms](https://codeminion.hashnode.dev/rate-limiting-algorithms#heading-solution)).

### Conclusion

Airlines/Trains also use similar algorithms and does a tentative allocation of the seat to the first user that expires after some length of time (e.g., 10 minutes for kiosks) that gives him enough time to pay. If the (customer-visible) transaction fails through or times out, the seat allocation can be released back into the pool. (All state changes are processed via the transactional database, and one customer-visible transaction might require many database-level transactions.)
 