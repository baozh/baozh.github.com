---
layout: post
title: "Quicksort in Linked List"
date: 2014-10-05 08:20
comments: true
categories: Software_development C/C++
tags: quicksort list 
---


Quick sort is a common sort method for objects stored contiguously, such as arrays. Recently, while implement a list template, I trying to migrate quick sort method to doubly linked list. The basic ideas of implementing quicksort in arrays and in lists are same, also based on partition, but there's one thing  different: The list does not support random access based on index. This essay will discuss how to implement quicksort in the list.

<!--more-->

### Swap List Nodes

When comes to sorting, it will involve the exchange of data between two objects in sequence to be ordered. In the list, this is the exchange of data between two list nodes. There are two methods for swaping list nodes.

1. By Swaping Data of Nodes

   This is the simplest, most common method: Exchange datas stored in list nodes. C/C++ codes for `swap` function like this:

		template <class T>
		void List<T>::Swap(T *data1, T *data2)
		{
			if (*data1 == *data2) return; 
			T tmp = *data1;
			*data1 = *data2;
			*data2 = tmp;
		}

2. By Manipulating Pointers of Nodes in List

   Sometimes, The cost of copying data of nodes is large. In this circumstance, we can operate pointers of nodes to achieve exchange of two nodes. C/C++ codes for `swap` function like this:

		template <class T>
		BOOL List<T>::Swap(ListNode<T> *pos1, ListNode<T> *pos2)
		{
			if (pos1 == NULL || pos2 == NULL)
			{
				return FALSE;
			}
			if(pos1 == pos2)
			{
				return TRUE;
			}

			ListNode<T> *temp, *temp1; 
			if(pos1 == m_pHead && pos2 == m_pTail)  /* swap the head node and the tail node */ 
			{ 
				if(pos1->m_pNext==pos2)/*only two nodes in list*/ 
				{ 
					pos2->m_pNext=pos1; 
					pos2->m_pPrev=NULL; 
					pos1->m_pPrev=pos2; 
					pos1->m_pNext=NULL; 
					m_pHead = pos2; 
					m_pTail = pos1;
				} 
				else
				{ 
					pos1->m_pNext->m_pPrev=pos2; 
					pos2->m_pPrev->m_pNext=pos1; 
					pos2->m_pNext=pos1->m_pNext; 
					pos1->m_pPrev=pos2->m_pPrev; 
					pos2->m_pPrev = pos1->m_pNext = NULL; 
					m_pHead = pos2;   /* assign new head node */ 
					m_pTail = pos1;   /* assign new tail node */ 
				}
			}
			else if(pos2 == m_pTail)    /* swap the tail node and any other node */  
			{ 
				if(pos1->m_pNext==pos2) /* swap the last two nodes */ 
				{ 
					pos1->m_pPrev->m_pNext=pos2;
					pos2->m_pPrev=pos1->m_pPrev;
					pos2->m_pNext=pos1; 
					pos1->m_pPrev=pos2; 
					pos1->m_pNext=NULL;
					m_pTail = pos1;    /* assign new tail node */ 
				} 
				else
				{ 
					temp=pos2->m_pPrev; 
					temp->m_pNext=pos1; 
					pos1->m_pPrev->m_pNext=pos2; 
					pos1->m_pNext->m_pPrev=pos2; 
					pos2->m_pPrev=pos1->m_pPrev; 
					pos2->m_pNext=pos1->m_pNext; 
					pos1->m_pPrev=temp; 
					pos1->m_pNext=NULL; 
					m_pTail = pos1;  /* assign new tail node */ 
				} 
			} 
			else if(pos1 == m_pHead) /* swap the head node and any other node */ 
			{ 
				if(pos1->m_pNext==pos2) /* swap the first two nodes */ 
				{ 
					pos2->m_pNext->m_pPrev=pos1; 
					pos1->m_pNext=pos2->m_pNext; 
					pos1->m_pPrev=pos2; 
					pos2->m_pNext=pos1; 
					pos2->m_pPrev=NULL; 
					m_pHead=pos2;     /* assign new head node */
				} 
				else
				{ 
					temp=pos1->m_pNext; 
					temp->m_pPrev=pos2; 
					pos1->m_pPrev=pos2->m_pPrev; 
					pos1->m_pNext=pos2->m_pNext; 
					pos2->m_pPrev->m_pNext=pos1; 
					pos2->m_pNext->m_pPrev=pos1; 
					pos2->m_pNext=temp; 
					pos2->m_pPrev=NULL; 
					m_pHead=pos2;    /* assign new head node */ 
				} 
			} 
			else /* swap any other two nodes except the head and tail node */ 
			{ 
				if(pos1->m_pNext==pos2) /* swap two adjacent nodes */ 
				{ 
					temp=pos1->m_pPrev; 
					pos1->m_pPrev->m_pNext=pos2; 
					pos1->m_pNext->m_pPrev=pos2; 
					pos1->m_pPrev=pos2; 
					pos1->m_pNext=pos2->m_pNext; 
					pos2->m_pNext->m_pPrev=pos1; 
					pos2->m_pNext=pos1; 
					pos2->m_pPrev=temp; 
				} 
				else
				{ 
					temp1 = pos1->m_pPrev;
					pos1->m_pPrev->m_pNext=pos2; 
					pos1->m_pNext->m_pPrev=pos2; 
					pos1->m_pPrev=pos2->m_pPrev; 

					temp=pos1->m_pNext;
					pos1->m_pNext=pos2->m_pNext; 

					pos2->m_pPrev->m_pNext=pos1; 
					pos2->m_pNext->m_pPrev=pos1; 
					pos2->m_pNext=temp; 

					pos2->m_pPrev=temp1; 
				} 
			} 
			return TRUE;
		}; 
	
	Function Description:  
	- According to the position of two nodes, pointer operation has been processed for different circumstances.
	- In this function, parameter `pos1` must be located in front of `pos2`. 
		
### Implementation Quicksort in List

Quicksort is a classic sort algorithm, the main step[[refer wikipedia][link1]] of sorting is:

1. Pick an element, called a **pivot**, from the array.
2. Reorder the array so that all elements with values less than the pivot come before the pivot, while all elements with values greater than the pivot come after it (equal values can go either way). After this partitioning, the pivot is in its final position. This is called the **partition** operation.
3. Recursively apply the above steps to the sub-array of elements with smaller values and separately to the sub-array of elements with greater values.

Applying above steps in list has serveral difference from applying in an array. The list doesn't support random access, can't random access the K-th element, so select the first node as pivot every time when sort sub-sequence objects.

#### Implementation Steps

The list to be sorted is split into two sub-lists. For simplicity, select the first node as a pivot, and then comparing all the nodes with the pivot, pick the nodes with smaller values than pivot to the left sub-list, pick the nodes with greater values than pivot to the right sub-list. After that, insert the pivot into list, as a bridge connecting the two sub-lists. Finally, recursively apply these steps to the left sub-list, right sub-list. 

There is a little trick in `partition` step. In fact, Implementing `partition` just need traverse once the linked list. The methods is: Define two pointers `pslow`, `pfast`, `pslow` initialized to the head node of the list, `pfast` initialized to the next node after the head node; Using `pslow` as pointer to the last node of sub-list with smaller values than pivot , using `pfast` traverse the list. Every time encounter a smaller node than pivot, move `pslow` to the next node, then swap the `pslow` node and `pfast` node.

1. Swap Data of Nodes Version

   This version of quicksort is using swap method by swaping data of nodes. C/C++ codes for `quicksort` function like this:

		template <class T>
		void List<T>::__sort_list(ListNode<T> *begin, ListNode<T> *end)  
		/* begin pointer to the fisrt node of list, end pointer to the last node of list */
		{
			if(begin == NULL || end == NULL)
				return;  
			if(begin == end)  
				return;  
			ListNode<T> *pslow = begin;  
			ListNode<T> *pfast = begin->m_pNext;  
			while(true)  
			{  
				if(pfast->m_tData < begin->m_tData)            /* select the first node as a pivot */
				{   
					pslow = pslow->m_pNext;
					/* pslow always pointer to the last node of sub-list with smaller values than pivot */
					Swap(&pslow->m_tData, &pfast->m_tData);    
				}  
				pfast = pfast->m_pNext;
			}  
			/* pslow pointer to the proper position of list which pivot should stored at this moment */
			Swap(&pslow->m_tData , &begin->m_tData);  
			/* recursively apply __sort_list function to the left sub-list, right sub-list */
			__sort_list(begin, pslow);
			__sort_list(pslow->next, end);
		};

2. Manipulate Pointers of Nodes Version

   This version use swap method by manipulating pointers of nodes in List. C/C++ codes for `quicksort` function like this:

		template <class T>
		void List<T>::__swap_ptr(ListNode<T> **pos1, ListNode<T> **pos2)   /* swap two pointer of list nodes */
		{
			if (*pos1 == *pos2)
			{
				return;
			}
			ListNode<T> *tmp = *pos1;
			*pos1 = *pos2;
			*pos2 = tmp;
		};   
   
		template <class T>
		void List<T>::__sort_list(ListNode<T> *begin, ListNode<T> *end)
		{
			if(begin == NULL || end == NULL)  
				return;  
			if(begin == end)  
				return;  
			ListNode<T> *pslow = begin;  
			ListNode<T> *pfast = begin->m_pNext;  
			ListNode<T> *ptemp = NULL, *ptemp1 = NULL, *pEndFlag = NULL;
			pEndFlag = end;
			while(true)  
			{  
				ptemp = pfast;
				ptemp1 = pfast->m_pNext;
				if(pfast->m_tData < begin->m_tData)    /* select the first node as a pivot */  
				{   
					pslow = pslow->m_pNext;
					/* pslow always pointer to the last node of sub-list with smaller values than pivot */
					Swap(pslow, pfast);                
					__swap_ptr(&pslow, &pfast);   /* attention! */
				}  
				if (ptemp == pEndFlag || ptemp == NULL)
				{
					break;
				}
				pfast = ptemp1;
			}  
			
			BOOL32 bIsPivotLeft = FALSE, bIsPivotRight = FALSE;
			if (pslow == begin)
			{
			/* the proper postion pivot stored is leftmost of the list, so there is no need to sort left sub-list */  
				bIsPivotLeft = TRUE;        
			}
			if (pslow == end)
			{
			/* the proper postion pivot stored is rightmost of the list, so there is no need to sort right sub-list */ 
				bIsPivotRight = TRUE;      
				end = begin;
			}
			/* pslow pointer to the proper position of list which pivot should stored at this moment */
			Swap(pslow, begin);              
			__swap_ptr(&pslow, &begin);      /* attention! */
			
			/* recursively apply __sort_list function to the left sub-list, right sub-list */
			ListNode<T> *pLeftEnd = pslow->m_pPrev;
			ListNode<T> *pRightBegin = pslow->m_pNext;
			if (bIsPivotLeft == FALSE)
			{
				__sort_list(begin, pLeftEnd); 
			}
		 
			if (bIsPivotRight == FALSE)
			{
				__sort_list(pRightBegin, end);
			}
		};

**Points for Attention**  

`Swap` function just operates pointers inside list nodes, such as `m_pPrev`, `m_pNext`, it dosen't modify the value of two nodes pointers passed by function. It meams `pos1`,`pos2` still pointer to the nodes with origin value. Therefore, after calling `Swap`, we must swap two pointers' value using `__swap_ptr` function.


I implemented quicksort in doubly linked list, you can see C/C++ codes in [file][link2].


[link1]: http://en.wikipedia.org/wiki/Quicksort
[link2]: https://github.com/baozh/container/blob/master/include/list.h







