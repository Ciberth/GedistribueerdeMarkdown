# Labo 3: WCF

## Project BadmintonClientConsole

### Program.cs

```cs

namespace BadmintonClientConsole
{
    class Program
    {
        static void Main(string[] args)
        {
            BadmintonServiceClient client = new BadmintonServiceClient();
            foreach (SportClub club in client.GetSportClubs())
            {
                Console.WriteLine(club.ID + " " + club.Naam);
                if (club.Tornooien.Length > 0)
                {
                    Console.WriteLine("Tornooien");
                    foreach (Tornooi tornooi in club.Tornooien)
                    {
                        Console.WriteLine(tornooi.Naam);
                    }
                }
            }

            Console.WriteLine("Geef id van club: ");
            string idString = Console.ReadLine();
            int id = int.Parse(idString);
            Console.WriteLine("Leden");
            foreach (Lid lid in client.GetLeden(id))
            {
                Console.WriteLine(lid.Naam);
            }
            client.Close();

        }
    }
}


```

## Project BadmintonServerConsole

### Program.cs

```cs

namespace BadmintonServerConsole
{
    class Program
    {
        static void Main(string[] args)
        {
            // ServiceHost aanmaken
            using (ServiceHost serviceHost =
                   new ServiceHost(typeof(BadmintonServiceLibrary.BadmintonService)))
            {
                // ServiceHost openen en luisteren naar berichten
                serviceHost.Open();

                // Service actief
                Console.WriteLine("De BadmintonServer is klaar.");
                Console.WriteLine("Klik <ENTER> om af te sluiten.");
                Console.WriteLine();
                Console.ReadLine();
            }

        }
    }
}


```


## BadmintonService

### IService1.cs 

```cs
namespace BadmintonService
{
    // NOTE: You can use the "Rename" command on the "Refactor" menu to change the interface name "IService1" in both code and config file together.
    [ServiceContract]
    public interface IService1
    {

        [OperationContract]
        string GetData(int value);

        [OperationContract]
        CompositeType GetDataUsingDataContract(CompositeType composite);

        // TODO: Add your service operations here
    }


    // Use a data contract as illustrated in the sample below to add composite types to service operations.
    [DataContract]
    public class CompositeType
    {
        bool boolValue = true;
        string stringValue = "Hello ";

        [DataMember]
        public bool BoolValue
        {
            get { return boolValue; }
            set { boolValue = value; }
        }

        [DataMember]
        public string StringValue
        {
            get { return stringValue; }
            set { stringValue = value; }
        }
    }
}


```

## BadmintonServiceLibrary

### IBadmintonService.cs

```cs

namespace BadmintonServiceLibrary
{
    // NOTE: You can use the "Rename" command on the "Refactor" menu to change the interface name "IService1" in both code and config file together.
    [ServiceContract]
    public interface IBadmintonService
    {
        [OperationContract]
        SportClub[] GetSportClubs();

        [OperationContract]
        Lid[] GetLeden(int sportclubID);
    }

    [DataContract]
    public class SportClub
    {
        [DataMember]
        public int ID { get; set; }

        [DataMember]
        public String Naam { get; set; }

        [DataMember]
        public Tornooi[] Tornooien { get; set; }
    }

    [DataContract]
    public class Lid
    {
        [DataMember]
        public int ID { get; set; }

        [DataMember]
        public String Naam { get; set; }
    }

    [DataContract]
    public class Tornooi
    {
        [DataMember]
        public int ID { get; set; }

        [DataMember]
        public String Naam { get; set; }
    }

}

```

### BadmintonService.cs

```cs

namespace BadmintonServiceLibrary
{
    // NOTE: You can use the "Rename" command on the "Refactor" menu to change the class name "Service1" in both code and config file together.
    public class BadmintonService : IBadmintonService
    {
        private BadmintonInterface.BadmintonDAO dao;

        BadmintonService()
        {
            dao = new BadmintonInterface.BadmintonDAODummy();
        }

        public SportClub[] GetSportClubs()
        {
            BadmintonInterface.SportClub[] clubs = dao.SportClubs;
            return ConverteerSportClubs(clubs);
        }

        private SportClub[] ConverteerSportClubs(BadmintonInterface.SportClub[] clubs)
        {
            SportClub[] badmintonClubs = new SportClub[clubs.Length];
            for (int i = 0; i < badmintonClubs.Length; i++)
            {
                BadmintonInterface.SportClub club = clubs[i];
                SportClub badmintonClub = new SportClub();
                badmintonClub.ID = club.ID;
                badmintonClub.Naam = club.Naam;
                badmintonClub.Tornooien = ConverteerTornooien(club.Tornooien);
                badmintonClubs[i] = badmintonClub;
            }
            return badmintonClubs;
        }

        private Tornooi[] ConverteerTornooien(IList<BadmintonInterface.Tornooi> tornooien)
        {
            Tornooi[] webTornooien = new Tornooi[tornooien.Count];
            for (int i = 0; i < webTornooien.Length; i++)
            {
                BadmintonInterface.Tornooi tornooi = tornooien[i];
                Tornooi webTornooi = new Tornooi();
                webTornooi.ID = tornooi.ID;
                webTornooi.Naam = tornooi.Naam;
                webTornooien[i] = webTornooi;
            }
            return webTornooien;
        }


        public Lid[] GetLeden(int sportclubID)
        {
            BadmintonInterface.Lid[] leden = dao.GeefLeden(sportclubID);
            Lid[] webLeden = ConverteerLeden(leden);
            return webLeden;
        }

        private Lid[] ConverteerLeden(BadmintonInterface.Lid[] leden)
        {
            Lid[] webLeden = new Lid[leden.Length];
            for (int i = 0; i < webLeden.Length; i++)
            {
                BadmintonInterface.Lid lid = leden[i];
                Lid webLid = new Lid();
                webLid.ID = lid.ID;
                webLid.Naam = lid.Naam;
                webLeden[i] = webLid;
            }
            return webLeden;
        }
    }

    
}


```

+ todo webclient 