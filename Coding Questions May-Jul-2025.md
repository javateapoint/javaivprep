# 1. Find All Permutations of String 

```java

public class StringPermutations {

    public static void main(String[] args) {
        String input = "xyz";
        permute(input).stream().forEach(System.out::println);
    }


    public static List<String> permute(String input) {
        List<String> result = new ArrayList<>();
        backtrace(input.toCharArray(), 0, result);
        return result;
    }

    private static void backtrace(char[] charArray, int index, List<String> result) {
        if(index == charArray.length) {
            result.add(Arrays.toString(charArray));
        }
        for(int i = index; i< charArray.length; i++) {
            swap(charArray,i ,index);
            backtrace(charArray, index + 1, result);
            swap(charArray,i ,index);

        }

    }

    public static void swap(char arr[], int sourceIndex, int targetIndex) {
        char temp = arr[sourceIndex];
        arr[sourceIndex] = arr[targetIndex];
        arr[targetIndex] = temp;
    }
}




```


# 2. Maximum Subarray Problem

```java

public class PlaySix {

    public static void main(String[] args) {
        int arr[] = {2, 3, -8, 7, -1, 2, 3};
        System.out.println(maxSubArray(arr));  // prints 11

//        int maxSum = 0;
//
//        for(int i =0; i< arr.length; i++) {
//            int sum =0;
//            for(int j=i; j< arr.length; j++) {
//                sum += arr[j];
//                if(sum > maxSum) {
//                    maxSum = sum;
//                }
//            }
//        }
//
//        System.out.println(maxSum);
    }

    public static int maxSubArray(int[] arr) {
        int sum = arr[0];
        int maxSum = arr[0];

        for (int i = 1; i< arr.length; i ++) {
            sum = Math.max(arr[i], sum + arr[i]);
            maxSum = Math.max(sum, maxSum);
        }
        return maxSum;
    }


}


```


# 3. String Expansion (Run-Length Decoding) with “1”s

```java


public class PlayThree {


    public static void main(String[] args) {

        String str = "a2b2c3";
        //Out put: a11b11c111


        String output = Arrays.stream(str.split(""))
                .map(PlayThree::mapChar)
                .collect(Collectors.joining());

        System.out.println(output);

        String op = str.chars()
                .mapToObj(c -> Character.isDigit(c) ? "1".repeat(c - '0') : String.valueOf((char)c))
                .collect(Collectors.joining());

        System.out.println(op);

    }


    public static String mapChar (String c) {

        if(Character.isDigit(c.charAt(0))) {
            StringBuilder sb = new StringBuilder();
            int count = Integer.valueOf(c);
            for(int i =0; i < count; i++) {
                sb.append("1");
            }
            return sb.toString();
        }
        return c;

    }

}


```
