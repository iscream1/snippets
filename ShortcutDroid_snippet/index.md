## 1. Socket kommunikáció kliens és szerver között

A szervernél kisebb az esélye, hogy NAT-olva van, ezért célszerû azt befogni szervernek, és a kliensrõl hozzá csatlakozni az interneten keresztül, és byte streamet beolvasni/küldeni.

´´´server = new TcpListener(IPAddress.Any, port);
server.Start();
while (client == null || client.Connected == false)
{
	client=server.AcceptTcpClient();
}
stream = client.GetStream();
while ((i = stream.Read(bytes, 0, bytes.Length)) != 0)
{
	string[] dataArray=data.Split(new[] { "<sprtr>" }, StringSplitOptions.None);
	//initial setup needed when connecting
	if (dataArray[0]=="setup")
        {
        	setupInit();
        }
	//the user selected another app on the client
        else if (dataArray[0]=="selected")
        {
        	if (SpinnerSelectedEvent != null)
        		SpinnerSelectedEvent(dataArray[1]); //send the app specific shortcuts
        }
        //received a keystroke which should be processed and executed
        else if (dataArray[0]=="keystroke")
        {
        	wrapper.Send(dataArray[1]);
        }
}´´´

## 2. Billentyûlenyomások szimulálása kódból

Windows alatt van erre egy rendkívül jól megírt API, a [SendKeys](https://msdn.microsoft.com/en-us/library/system.windows.forms.sendkeys(v=vs.110).aspx). Erre írtam egy wrappert, ami a gyakran használt kifejezéseket leegyszerûsíti. A ´(seq)(/seq)´ tagek között lévõ szöveget alakítja át, és küldi ki egyszerre, billentyûkombinációként is. Pl: ´(seq)\s\f2(/seq)´ a Shift+F2 kombinációt eredményezi (´+{F2}´ SendKeys szintaxis szerint). Ha nem F billentyû áll a végén, akkor más karakter is állhat utána, pl ´\c\sf´ Ctrl+Shift+F lesz, de használható a SendKeys szintaxis is a seq tagen belül.

Seq tagen belüli szintaxis kezelõföggvény részlete:
´//key sequences must be sent together
private void SendFast(string toSend)
{
	StringBuilder output = new StringBuilder();

	//split before each escaped character
	string[] array = toSend.Split(new[] { "\\" }, StringSplitOptions.None);
	foreach (string s in array)
	{
		//if it's an empty string, there's no need to check
		//E.g. "\\n".Split(..."\\"...); will have an empty string as 0th member because
                //the first occurrence is at the beginning of the string
                if (s.Length != 0)
                    switch (s[0])
                    {
                        //convert to SendKeys semantics and append if it has more chars
                        case 's': //shift
                            output.Append("+"); output.Append(s.Substring(1)); break;
                        case 'c': //ctrl
                            output.Append("^"); output.Append(s.Substring(1)); break;
                        case 'a': //alt
                            output.Append("%"); output.Append(s.Substring(1)); break;
...
	SendKeys.SendWait(output.ToString());
}´

A seq tagen kívül lévõ karaktereket külön szimulálja a szerver, és közöttük random idõközt vár, hogy úgy tûnjön, mintha egy valódi ember gépelné. Ha csak billentyûkombinációt küldünk, akkor seq tagen belül adunk információt, így ez a billentyûkombinációkat nem érinti.

´//wait between each character, simulating a real person typing
private void SendSlow(string s)
{
	SendKeys.SendWait(s);
	Thread.Sleep(rand.Next(25, 100));
}´

Néhány karakternek a SendKeys szintaxison belül sajátos jelentése van, így azokat escapelni kell {} tagek közé. Ilyen a {, }, (, ), és még más kevésbé használtak. Ezek nyilvánvalóan a lassú szögvegnél jelentenek problémát, mivel billentyûkombinációban nem gyakran szerepel "{" **billentyû**.

´switch (c)
{
	//cases for single characters that need special wrapping in SendKeys
	case '{':
	{
		//these must be sent together
		SendFast("{{}");
	}
	break;
	case '}':
	{
		SendFast("{}}");
	}
	break;´