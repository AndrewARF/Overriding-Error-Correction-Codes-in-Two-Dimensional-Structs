The relentless pursuit of highly reliable communication systems demands research and development of algorithms capable of identifying
and correcting errors that may occur during data transmission and storage. Reliability is even more critical in critical systems and/or
those that are difficult to access, such as space, passenger transportation, and finance. In this scenario, the Error Correction Code (ECC) 
is a fundamental tool to provide a certain degree of reliability to systems. This work proposes a new technique to increase the error correction
capacity of ECCs based on region overlapping. More specifically, we propose correcting data areas protected by more than one ECC, allowing for
the inference of logic correlating ECCs, thus enhancing their error detection and correction capability. The work focuses on bidimensional 
codeword organizations, known as 2D-ECCs, which constitute a hierarchical arrangement of ECCs. The proposed overlapping approach will be
evaluated in some 2D organizations, comparing error correction and detection capabilities, scalability, and reliability.



public class CodeStruct {
	public int D[] = new int[16];	// Data array
	public int Co[] = new int[5];	// Outer check-bit array
	public int Ci[] = new int[5];	// Inner check-bit array
	public int Po, Pi; 				// Outer and Inner parity bits
	// Receive data array, codify in Hamming and parity for the Outer and Inner ECCs
	public CodeStruct(int D[]) throws Exception {
		for(int r=0; r<this.D.length; r++)
			this.D[r] = D[r];
		encodeOuterHamming(Co);
		Po = encodeHammingParity(Co);
		encodeInnerHamming(Ci);
		Pi = encodeHammingParity(Ci);
	}
	protected void encodeOuterHamming(int ham[]) { // Codify Outer Hamming
		ham[0] = D[11] ^ D[4]  ^ D[2]  ^ D[6]  ^ D[14] ^ D[8]  ^ D[12]; 	
		ham[1] = D[7]  ^ D[5]  ^ D[15] ^ D[10] ^ D[3]  ^ D[9]  ^ D[6] ^ D[14] ^ D[8] ^ D[12];
		ham[2] = D[13] ^ D[1]  ^ D[15] ^ D[10] ^ D[3]  ^ D[9]  ^ D[2] ^ D[12];
		ham[3] = D[0]  ^ D[1]  ^ D[5]  ^ D[3]  ^ D[9]  ^ D[4]  ^ D[2] ^ D[8];
		ham[4] = D[0]  ^ D[13] ^ D[1]  ^ D[7]  ^ D[5]  ^ D[10] ^ D[9] ^ D[11] ^ D[4] ^ D[2] ^ D[14] ^ D[12];
	}
	protected void encodeInnerHamming(int ham[]) { // Codify Inner Hamming
		ham[0]	= D[15] ^ D[10] ^ D[12] ^ D[9]  ^ D[3]  ^ D[11] ^ D[7]  ^ D[5] ^ D[2];
		ham[1]	= D[8]  ^ D[13] ^ D[1]  ^ D[4]  ^ D[9]  ^ D[3]  ^ D[11] ^ D[7] ^ D[5] ^ D[2];
		ham[2]	= D[6]  ^ D[0]  ^ D[1]  ^ D[4]  ^ D[12] ^ D[11] ^ D[7]  ^ D[5] ^ D[2];
		ham[3]	= D[14] ^ D[0]  ^ D[13] ^ D[15] ^ D[10] ^ D[12] ^ D[3]  ^ D[5] ^ D[2];
		ham[4]	= D[14] ^ D[6]  ^ D[8]  ^ D[4]  ^ D[10] ^ D[12] ^ D[9]  ^ D[3] ^ D[7] ^ D[2];
	}
	protected int encodeHammingParity(int ham[]) { // Codify Inner and Outer Parity
		int parity = ham[0];
		
		for(int k=1; k<ham.length; k++)
			parity =  parity ^ ham[k];
		for(int j=0; j<this.D.length; j++) {
			parity = parity ^ D[j];
		}
		return parity;
	}
	public boolean isEqual(CodeStruct ecc) { // Compare data array of two instances of Codestruct
		for(int k=0; k<D.length; k++) {
			if(D[k] != ecc.D[k])
				return false;
		}
		return true;
	}
	public String toString() {
		String str = "";
		str = str + "[" + D[0]  + " " + D[1]  + " " + D[2]  + " " + D[3]  + "][" + Co[0] + " " + Co[1] + " " + Co[2] + " " + Co[3] + " " + Co[4] + "][" + Po + "]\n";
		str = str + "[" + D[4]  + " " + D[5]  + " " + D[6]  + " " + D[7]  + "][" + Ci[0] + " " + Ci[1] + " " + Ci[2] + " " + Ci[3] + " " + Ci[4] + "][" + Pi + "]\n";
		str = str + "[" + D[8]  + " " + D[9]  + " " + D[10] + " " + D[11] + "]\n";
		str = str + "[" + D[12] + " " + D[13] + " " + D[14] + " " + D[15] + "]\n";
		return str;
	}
}




public class CodeStructWithErrror extends CodeStruct
{
	public int recCo[] = new int[5];	// Array storing the computation of the Outer Hamming in the decoding process 
	public int recCi[] = new int[5];	// Array storing the computation of the Inner Hamming in the decoding process 
	public int recPo, recPi;			// Bits storing the Outer and Inner parities computed in the decoding process 
	
	public int sCo[] = new int[5];		// Array of Outer Hamming syndromes
	public int sCi[] = new int[5];		// Array of Inner Hamming syndromes
	public int sPo, sPi;				// Outer and Inner parity syndromes
	public int sCoq, sCiq;				// Bits enclosing the OR of all bits of the Outer and Inner syndrome arrays

	public int EArO, EArI;				// Outer and Inner error addresses
	public boolean DErO, SErO;			// Flags of double and single error on Outer ECC
	public boolean DErI, SErI;			// Flags of double and single error on Inner ECC
	public CodeStructWithErrror(CodeStruct initialCodeStruct, int errorPattern[]) throws Exception { // Constructor for creating an instance of the code with an error pattern inserted
		super(initialCodeStruct.D);
		setErrorPattern(errorPattern);
		recomputeControlVariables();
	}
	public void recomputeControlVariables() { // Method that recompute all control variables after correcting an error
		recomputeCheckBitsAndParity();
		computeSyndromes();
		computeErrorAddress();
		computeSE_DE();
	}
	public void recomputeCheckBitsAndParity() { // Recompute Hamming and parity of Outer and Inner ECCs
		encodeOuterHamming(recCo);
		recPo = encodeHammingParity(Co);
		encodeInnerHamming(recCi);
		recPi = encodeHammingParity(Ci);
	}
	public void computeSyndromes() { // Compute all syndromes
		for(int k=0; k<sCi.length; k++)
			sCi[k] = Ci[k] == recCi[k] ? 0 : 1;
		sPi = Pi == recPi ? 0 : 1;
		
		for(int k=0; k<sCo.length; k++)
			sCo[k] = Co[k] == recCo[k] ? 0 : 1;
		sPo = Po == recPo ? 0 : 1;
		
		sCiq = (sCi[0]==1 || sCi[1]==1 || sCi[2]==1 || sCi[3]==1 || sCi[4]==1) ? 1 : 0;
		sCoq = (sCo[0]==1 || sCo[1]==1 || sCo[2]==1 || sCo[3]==1 || sCo[4]==1) ? 1 : 0;
	}
	public void computeErrorAddress() {	// Compute outer and inner error address
		EArO = sCo[0] * 16 + sCo[1] * 8 + sCo[2] * 4 + sCo[3] * 2 + sCo[4];
		EArI = sCi[0] * 16 + sCi[1] * 8 + sCi[2] * 4 + sCi[3] * 2 + sCi[4];
	}
	public void computeSE_DE() {	// Compute single and double error
		SErI = sCiq==1 && sPi==1;
		DErI = sCiq==1 && sPi==0;
		SErO = sCoq==1 && sPo==1;
		DErO = sCoq==1 && sPo==0;
	}
	private void setErrorPattern(int errorPattern[]) throws Exception {
		for(int k = 0; k < errorPattern.length; k++)
			setError(errorPattern[k]);
	}
	private static int invertBit(int value) {
		return value == 0 ? 1 : 0;
	}
	private void setError(int errorPosition) throws Exception {
		if(errorPosition < 16)
			D[errorPosition] = invertBit(D[errorPosition]);
		else if(errorPosition >= 16 && errorPosition < 21) {
			errorPosition -= 16;
			Co[errorPosition] = invertBit(Co[errorPosition]);
		}
		else if(errorPosition == 21) {
			Po = invertBit(Po);
		}
		else if(errorPosition >= 22 && errorPosition < 27) {
			errorPosition -= 22;
			Ci[errorPosition] = invertBit(Co[errorPosition]);
		}
		else if(errorPosition == 27) {
			Pi = invertBit(Pi);
		}
		else
			throw new Exception("Error position = " + errorPosition);
	}
	public String toString() {
		String str = "D [0 1 2 3]  [0 1 2 3 4][P]   [0 1 2 3 4][sCq][sP][DE][SE][Add]\n";
		
		int de = DErO == true ? 1 : 0;
		int se = SErO == true ? 1 : 0;
		int di = DErI == true ? 1 : 0;
		int si = SErI == true ? 1 : 0;
		str = str+" 0["+D[0] +" "+D[1] +" "+D[2] +" "+D[3] +"]Ce["+Co[0]+" "+Co[1]+" "+Co[2]+" "+Co[3]+" "+Co[4]+"]["+Po+"]sCe["+sCo[0]+" "+sCo[1]+" "+sCo[2]+" "+sCo[3]+" "+sCo[4]+"]  ["+sCoq+"] ["+sPo+"] ["+de+"] ["+se+"]["+EArO+"]\n";
		str = str+" 1["+D[4] +" "+D[5] +" "+D[6] +" "+D[7] +"]Ci["+Ci[0]+" "+Ci[1]+" "+Ci[2]+" "+Ci[3]+" "+Ci[4]+"]["+Pi+"]sCi["+sCi[0]+" "+sCi[1]+" "+sCi[2]+" "+sCi[3]+" "+sCi[4]+"]  ["+sCiq+"] ["+sPi+"] ["+di+"] ["+si+"]["+EArI+"]\n";
		str = str+" 2["+D[8] +" "+D[9] +" "+D[10]+" "+D[11]+"]\n";
		str = str+" 3["+D[12]+" "+D[13]+" "+D[14]+" "+D[15]+"]\n";
		
		return str;
	}
}



public class Decoder { 
// 5-bit Hamming code covers up to 26-bit data; 16 bits are used with array addresses, the remaining receive -1
// 									     0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
	private static int outerAddTab[] = {-1,-1,-1, 0,-1,13,-1, 1,-1, 7,-1, 5,15,10, 3, 9,-1,11,-1, 4,-1,-1,-1, 2, 6,14, 8,-1,-1,12,-1,-1}; 
	private static int innerAddTab[] = {-1,-1,-1,14,-1, 6, 0,-1,-1, 8,13,-1, 1, 4,-1,-1,-1,-1,15,10,-1,-1,-1,12,-1, 9,-1, 3,11, 7, 5, 2};

// Outer Hamming Numbering						//	Inner Hamming Numbering
//		3	7	23	14	16	8	4	2	1		//	6	12	31	27	16	8	4	2	1
//		19	11	24	9							//	13	30	5	29
//		26	15	13	17							//	9   25	19	28
//		29	5	25	12							//	23	10	3	18
									
	private static final int[][][] doubleErrorMap = {
		//																																							EArI
		//            0         1         2         3         4         5         6         7         8         9         10        11        12        13        14        15        16        17        18        19        20        21        22        23        24        25        26        27        28        29        30        31
		/*  0 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/*  1 */ {{-1, -1}, {10, 15}, { 3,  9}, {-1, -1}, {-1, -1}, {-1, -1}, { 6, 14}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/*  2 */ {{-1, -1}, {-1, -1}, {-1, -1}, {5,   7}, {-1, -1}, {-1, -1}, { 1, 13}, {-1, -1}, {-1, -1}, { 3, 15}, { 9, 10}, {-1, -1}, { 6,  8}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4, 11}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/*  3 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 3, 10}, {-1, -1}, { 8, 14}, { 9, 15}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/*  4 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 5,  9}, {-1, -1}, {-1, -1}, { 0,  1}, {-1, -1}, {-1, -1}, {-1, -1}, { 7, 10}, {-1, -1}, {-1, -1}, {-1, -1}, { 2,  4}, {-1, -1}, {12, 14}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/*  5 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 3,  5}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {7 , 15}, {-1, -1}, {-1, -1}, { 6, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/*  6 */ {{-1, -1}, {-1, -1}, {-1, -1}, {2,  11}, { 7,  9}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 0, 13}, { 5, 10}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/*  7 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 3,  7}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 5, 15}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 8, 12}, {-1, -1}},
		/*  8 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1,  9}, {-1, -1}, {-1, -1}, { 0,  5}, {10, 13}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {11, 14}},
		/*  9 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4,  8}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1,  3}, {13, 15}, {6,  11}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 10 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 2, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4, 14}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 9, 13}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 0,  7}, {-1, -1}, {-1, -1}, {-1, -1}, { 1, 10}},
		/* 11 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4,  6}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 3, 13}, {-1, -1}, {-1, -1}, {-1, -1}, { 8, 11}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1, 15}, {-1, -1}},
/*E*/	/* 12 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {11, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1,  5}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 7, 13}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 0,  9}},
/*A*/	/* 13 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 2,  8}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 0,  3}, {-1, -1}, {-1, -1}},
/*r*/	/* 14 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1,  7}, {-1, -1}, {-1, -1}, { 5, 13}, { 0, 10}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4, 12}, {-1, -1}, { 2, 14}, {-1, -1}, {-1, -1}, {-1, -1}},
/*O*/	/* 15 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 0, 15}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 2,  6}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 16 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {10, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 0,  4}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1,  2}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 7, 14}, {-1, -1}},
		/* 17 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {12, 15}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 5,  8}, { 6,  7}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 18 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 9, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 2, 13}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 0, 11}, {-1, -1}, {-1, -1}, { 5, 14}, {-1, -1}, {-1, -1}},
		/* 19 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 3, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 7,  8}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 5,  6}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 20 */ {{-1, -1}, { 1,  4}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 7, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {10, 14}, {-1, -1}, { 3,  8}, {-1, -1}, {-1, -1}, {-1, -1}, {11, 13}, { 6, 15}, {-1, -1}, { 0,  2}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 21 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 8,  9}, {14, 15}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 6, 10}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 22 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4, 13}, {-1, -1}, { 5, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1, 11}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 9, 14}, { 8, 15}, {-1, -1}, {-1, -1}, { 3,  6}, {-1, -1}},
		/* 23 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 3, 14}, {-1, -1}, { 8, 10}, {-1, -1}, { 6,  9}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 24 */ {{-1, -1}, { 7, 11}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 2,  9}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4,  5}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {12, 13}, {-1, -1}, {-1, -1}},
		/* 25 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 2,  3}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 0,  8}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 26 */ {{-1, -1}, {-1, -1}, { 5, 11}, {-1, -1}, {-1, -1}, { 0, 14}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 2, 10}, {-1, -1}, {-1, -1}, {-1, -1}, { 4,  7}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 27 */ {{-1, -1}, {-1, -1}, {-1, -1}, { 0,  6}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 2, 15}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 28 */ {{-1, -1}, { 2,  5}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {13, 14}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {10, 11}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4,  9}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 29 */ {{-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1,  8}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {11, 15}, { 6, 13}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 3,  4}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}},
		/* 30 */ {{-1, -1}, {-1, -1}, { 2,  7}, {-1, -1}, {-1, -1}, { 9, 11}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 1, 14}, {-1, -1}, { 0, 12}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4, 10}, {-1, -1}},
		/* 31 */ {{-1, -1}, {-1, -1}, {-1, -1}, { 8, 13}, {-1, -1}, {-1, -1}, {-1, -1}, { 3, 11}, {-1, -1}, { 1,  6}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, {-1, -1}, { 4, 15}}
	};	
	public static void decoding(CodeStructWithErrror ecc) {
		if(ecc.EArO == 0 || ecc.EArI == 0)
			return;
		if(ecc.SErO) {
			int add = outerAddTab[ecc.EArO];
			if(add!=-1)
				ecc.D[add] = ecc.D[add]==0 ? 1 : 0; // Flip a bit
			ecc.recomputeControlVariables();
			return;
		}
		if(ecc.SErI) {
			int add = innerAddTab[ecc.EArI];
			if(add!=-1)
				ecc.D[add] = ecc.D[add]==0 ? 1 : 0; // Flip a bit
			ecc.recomputeControlVariables();
			return;
		}
		if(ecc.DErO && ecc.DErI) {
			int[] errorPositions = doubleErrorMap[ecc.EArO][ecc.EArI];
			int addA = errorPositions[0];
			int addB = errorPositions[1];

			if(addA != -1 && addB != -1) {
				ecc.D[addA] = ecc.D[addA]==0 ? 1 : 0; // Flip a bit
				ecc.D[addB] = ecc.D[addB]==0 ? 1 : 0; // Flip a bit
			}
			ecc.recomputeControlVariables();
			return;
		}
	}
}

public class MainSystem {
	public static final boolean DEBUG = false;	// Set true for debugging purposes
	
	private static int numErrors, initialElement = 0, numElements = 28;
	private static long numberOfDecodigns = 0, errorsAfterDecoding = 0;
	private static CodeStruct initialCodeStruct;
	private static CodeStructWithErrror eccWithErrors;

	private static void errorGenerator(int errorIndex, int errorPattern[], int elementIndex) throws Exception 
	{ 
		if(errorIndex == numErrors) 
		{ 
			eccWithErrors = new CodeStructWithErrror(initialCodeStruct, errorPattern);
			if(DEBUG){
				System.out.print(eccWithErrors);
			}
			Decoder.decoding(eccWithErrors);
			numberOfDecodigns++;
			
			if(!initialCodeStruct.isEqual(eccWithErrors)) {
				errorsAfterDecoding++;
				if(DEBUG){
					System.out.println(eccWithErrors);
					System.out.println("==> ERRO!\n");
				}
			}
			return; 
		} 
		if(elementIndex >= numElements) 
			return;
		errorPattern[errorIndex] = elementIndex; 
		errorGenerator(errorIndex+1, errorPattern, elementIndex+1); 
		errorGenerator(errorIndex, errorPattern, elementIndex+1); 
	}
	private static void setInitialCodeStruct() throws Exception {
		int D[] = {	0, 0, 0, 0, 
					0, 0, 0, 0,
					0, 0, 0, 0,
					0, 0, 0, 0, };
		initialCodeStruct = new CodeStruct(D);
	}
	private static void printTestIdentification() {
		System.out.println("#Errors=" + numErrors);
	}
	private static void printResults() {
	    System.out.println("\n\tNumberOfDecodigns = " + numberOfDecodigns);
	    System.out.println("\tNumberOfErrorsAfterDecoding = " + errorsAfterDecoding);
	}
	private static void setNumberOfErrors(int nE) {
		numErrors = nE;
	}
	private static void resetSimulationData() {
		numberOfDecodigns = 0;
		errorsAfterDecoding = 0;
	}
	private static void setErrorInterval(int inicio, int fim) {
		initialElement = inicio;
		numElements = fim;
	}
	public static void main(String[] args) throws Exception {
		long startTime = System.nanoTime();
		setInitialCodeStruct();
		setErrorInterval(0, 28);  // Default is (0, 28) ==> All bits
 		for(int numberOfErrors=0; numberOfErrors<=5; numberOfErrors++)
		{
			setNumberOfErrors(numberOfErrors);
			printTestIdentification();
			resetSimulationData();
			errorGenerator(0, new int[numErrors], initialElement);
			printResults();
		}
		long endTime = System.nanoTime();
		long duration = endTime - startTime;
		System.out.println("Duration: " + duration + " ns");
	}
}
